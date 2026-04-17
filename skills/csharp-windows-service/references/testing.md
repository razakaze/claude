# Testing

## What to test

- **Worker logic** — the `ExecuteAsync` body's decisions: retry, skip, transition, emit.
- **Options validation** — `required` members, range checks, cross-field constraints.
- **DI wiring** — one integration test per host to catch misregistrations.

## What not to test

- The hosting framework itself (`BackgroundService` lifecycle, SCM integration).
- `sc.exe`, the installer, or anything that requires admin on the build agent.
- Logging output text. Assert that *something* was logged at a given level, not the exact message.

## Worker unit tests

Don't build a host. Call `StartAsync`, let it run briefly, call `StopAsync`, assert.

```csharp
using FluentAssertions;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;
using NSubstitute;
using Xunit;

public class DataPollingWorkerTests
{
    [Fact]
    public async Task ContinuesAfterTransientFailure()
    {
        var source = Substitute.For<IDataSource>();
        source.PollAsync(Arg.Any<CancellationToken>()).Returns(
            Task.FromException(new HttpRequestException("transient")),
            Task.CompletedTask,
            Task.CompletedTask);

        var options = Options.Create(new PollingOptions
        {
            Interval = TimeSpan.FromMilliseconds(5)
        });
        var worker = new DataPollingWorker(
            NullLogger<DataPollingWorker>.Instance, source, options);

        using var cts = new CancellationTokenSource();
        await worker.StartAsync(cts.Token);
        await Task.Delay(50);
        cts.Cancel();
        await worker.StopAsync(CancellationToken.None);

        await source.Received().PollAsync(Arg.Any<CancellationToken>());
        source.ReceivedCalls().Count().Should().BeGreaterThan(1);
    }
}
```

Why `Task.Delay(50)`: lets `PeriodicTimer` tick a few times. Hacky but works for simple loops.

## PeriodicTimer in tests

The hack above isn't deterministic. For real timer control, use `TimeProvider` (available in .NET 8+). Inject `TimeProvider` into the worker and use `timeProvider.CreateTimer` or the `TimeProvider` overload of `PeriodicTimer`:

```csharp
public sealed class DataPollingWorker(
    ILogger<DataPollingWorker> logger,
    IDataSource source,
    IOptions<PollingOptions> options,
    TimeProvider timeProvider) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var timer = new PeriodicTimer(options.Value.Interval, timeProvider);
        while (await timer.WaitForNextTickAsync(stoppingToken))
        {
            await source.PollAsync(stoppingToken);
        }
    }
}
```

In production, register `TimeProvider.System` as a singleton:

```csharp
builder.Services.AddSingleton(TimeProvider.System);
```

In tests, use `Microsoft.Extensions.TimeProvider.Testing` (package name `Microsoft.Extensions.TimeProvider.Testing`):

```csharp
var time = new FakeTimeProvider();
var worker = new DataPollingWorker(logger, source, options, time);

await worker.StartAsync(CancellationToken.None);
time.Advance(TimeSpan.FromSeconds(1));
await Task.Yield();  // let the timer tick propagate

await source.Received(1).PollAsync(Arg.Any<CancellationToken>());

time.Advance(TimeSpan.FromSeconds(1));
await Task.Yield();

await source.Received(2).PollAsync(Arg.Any<CancellationToken>());
```

Deterministic. No sleep-based tests.

## Integration test

One test per service project, wiring the real host.

```csharp
[Fact]
public async Task HostStartsWithRealConfiguration()
{
    var builder = Host.CreateApplicationBuilder();
    builder.Configuration.AddInMemoryCollection(new Dictionary<string, string?>
    {
        ["Polling:Interval"] = "00:00:05",
        ["Polling:ConnectionString"] = "Data Source=:memory:",
        ["Polling:ServiceEndpoint"] = "http://localhost:1234",
    });

    builder.Services.AddSingleton(Substitute.For<IDataSource>());
    builder.Services.Configure<PollingOptions>(builder.Configuration.GetSection("Polling"));
    builder.Services.AddHostedService<DataPollingWorker>();

    using var host = builder.Build();
    await host.StartAsync();
    await Task.Delay(100);
    await host.StopAsync();
}
```

If this test passes, DI is wired, config binds, the worker starts. If it throws, you have a registration problem — easier to catch here than in prod.

## Logging assertions

Don't match message strings. Assert on level and event ID (if you use them).

```csharp
logger.Received().Log(
    LogLevel.Error,
    Arg.Any<EventId>(),
    Arg.Any<object>(),
    Arg.Any<HttpRequestException>(),
    Arg.Any<Func<object, Exception?, string>>());
```

NSubstitute's `ILogger` extension helpers make this cleaner — use `logger.Received().LogError(...)` patterns if you have the helpers. Otherwise the verbose form above.

For behavior-driven tests, prefer asserting on side effects (mock was called, state changed) over log assertions entirely. Logs are observability, not contract.

## Options validation

Test the options class, not the worker:

```csharp
[Fact]
public void PollingOptions_Fails_WhenRequiredMissing()
{
    var services = new ServiceCollection();
    services.AddOptions<PollingOptions>()
        .BindConfiguration("Polling")
        .ValidateDataAnnotations()
        .ValidateOnStart();

    // ... build provider, attempt to resolve, assert exception
}
```

Or just construct the options type and let compile-time `required` enforcement do its job. If you can instantiate it without setting the required fields, the required modifier is missing.

## What a good test suite looks like

- One unit test class per `BackgroundService`, ~5-10 tests.
- One integration test per host, ~1 test.
- Zero tests that call `sc.exe` or touch the installed service.
- Zero tests that sleep more than 200ms.
- Running the whole suite takes under 10 seconds.

If any of those numbers is off, the test design is probably wrong.
