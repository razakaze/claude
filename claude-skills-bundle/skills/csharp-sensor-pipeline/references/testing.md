# Testing

No test touches real hardware. Hardware is validated separately in on-device smoke tests. Everything here runs in-process on the build agent.

## Stage unit tests

Each stage is an interface method; mock the next stage; feed synthetic input.

```csharp
public class TemperatureProtocolStageTests
{
    [Fact]
    public async Task ParsesValidFrame()
    {
        var next = Substitute.For<INormalizationStage>();
        var stage = new TemperatureProtocolStage(next,
            NullLogger<TemperatureProtocolStage>.Instance);

        var frame = Convert.FromHexString("aa0123047f00");
        await stage.ProcessAsync(frame, CancellationToken.None);

        await next.Received(1).ProcessAsync(
            Arg.Is<RawTemperatureReading>(r => r.SensorId == 0x0123 && r.RawValue == 1151),
            Arg.Any<CancellationToken>());
    }

    [Fact]
    public async Task DropsFrameWithBadChecksum()
    {
        var next = Substitute.For<INormalizationStage>();
        var stage = new TemperatureProtocolStage(next,
            NullLogger<TemperatureProtocolStage>.Instance);

        await stage.ProcessAsync(Convert.FromHexString("aa0123047f01"), default);

        await next.DidNotReceive().ProcessAsync(
            Arg.Any<RawTemperatureReading>(), Arg.Any<CancellationToken>());
    }
}
```

Assert drops by asserting `DidNotReceive`. Never assert on log output.

## Stateful stage tests (stage 4)

Alarm stages read and write the context cache. Use a real cache.

```csharp
public class HighTemperatureAlarmStageTests
{
    [Fact]
    public async Task FiresAfterThreeConsecutiveHighReadings()
    {
        var next = Substitute.For<IFinalStage>();
        var cache = new ContextCache("s1");
        cache.Put("alarm.rules", new AlarmRule[] { new HighThreshold(80) });

        var stage = new HighTemperatureAlarmStage(cache, next,
            NullLogger<HighTemperatureAlarmStage>.Instance);

        await stage.ProcessAsync(Reading(81), default);
        await stage.ProcessAsync(Reading(82), default);
        await stage.ProcessAsync(Reading(83), default);

        await next.Received(3).ProcessAsync(
            Arg.Any<SensorEvent>(), Arg.Any<CancellationToken>());
        await next.Received(1).ProcessAsync(
            Arg.Is<SensorEvent>(e => e.Annotations.Any(a => a.Code == "HIGH_TEMP")),
            Arg.Any<CancellationToken>());
    }

    private static CanonicalReading Reading(double celsius) =>
        new("s1", DateTimeOffset.UtcNow, celsius);
}
```

## Fake transport

For end-to-end pipeline tests, a fake `IIntegrationStage` that lets the test push frames:

```csharp
public sealed class FakeIntegrationStage(IProtocolStage next) : IIntegrationStage
{
    public Task StartAsync(CancellationToken ct) => Task.CompletedTask;
    public Task StopAsync(CancellationToken ct) => Task.CompletedTask;

    public ValueTask PushAsync(ReadOnlyMemory<byte> frame, CancellationToken ct)
        => next.ProcessAsync(frame, ct);
}
```

Test drives the pipeline by calling `PushAsync`.

## End-to-end pipeline test

Wire five real stages, push frames through the fake integration, collect events from an in-memory sink.

```csharp
public class PipelineEndToEndTests
{
    [Fact]
    public async Task AnnotatesEventWhenTemperatureExceedsThreshold()
    {
        var sink = new InMemorySensorEventSink();
        var cache = new ContextCache("s1");
        cache.Put("calibration.offset", 0.0);
        cache.Put("calibration.scale", 0.1);
        cache.Put("alarm.rules", new AlarmRule[] { new HighThreshold(80) });

        var final   = new FinalStage(sink, TestConfig(), NullLogger<FinalStage>.Instance);
        var alarm   = new HighTemperatureAlarmStage(cache, final, NullLogger<HighTemperatureAlarmStage>.Instance);
        var norm    = new TemperatureNormalizationStage(cache, alarm, NullLogger<TemperatureNormalizationStage>.Instance);
        var proto   = new TemperatureProtocolStage(norm, NullLogger<TemperatureProtocolStage>.Instance);
        var fake    = new FakeIntegrationStage(proto);

        await fake.PushAsync(FrameAt(850), default);    // 85.0°C after scale
        await fake.PushAsync(FrameAt(860), default);
        await fake.PushAsync(FrameAt(870), default);

        sink.Events.Should().HaveCount(3);
        sink.Events.Last().Annotations.Should().ContainSingle(a => a.Code == "HIGH_TEMP");
    }

    private static SensorConfig TestConfig() => /* ... */;
    private static byte[] FrameAt(int rawValue) => /* ... */;
}

internal sealed class InMemorySensorEventSink : ISensorEventSink
{
    public List<SensorEvent> Events { get; } = [];

    public ValueTask PublishAsync(SensorEvent evt, CancellationToken ct)
    {
        Events.Add(evt);
        return ValueTask.CompletedTask;
    }
}
```

## Config tests

In-memory SQLite plus YAML seeder.

```csharp
public class YamlSeederTests
{
    [Fact]
    public void SqliteWinsOnConflict()
    {
        using var conn = new SqliteConnection("Data Source=:memory:");
        conn.Open();
        SchemaMigrator.Migrate(conn);

        // Pre-insert a sensor that differs from YAML
        InsertSensor(conn, "s1", /* transport */ "udp", /* port */ 5000);

        var yaml = """
            sensors:
              - sensor_id: s1
                sensor_type: temperature
                enabled: true
                transport:
                  type: udp
                  port: 9999
                p99_budget_ms: 50
            """;

        var seeder = new YamlSensorSeeder(NullLogger<YamlSensorSeeder>.Instance);
        seeder.Seed(WriteTempFile(yaml), "Data Source=:memory:", reseed: false);  // use same conn somehow

        var sensor = LoadSensor(conn, "s1");
        var transport = System.Text.Json.JsonSerializer.Deserialize<UdpTransport>(sensor.Transport);
        transport!.Port.Should().Be(5000);  // SQLite won
    }
}
```

**Important:** when testing with in-memory SQLite, you must share the connection across seeder and assertion — `Data Source=:memory:` by default creates a fresh DB per connection. Use `Data Source=file::memory:?cache=shared` or pass the open connection directly.

## Time-dependent tests

Inject `TimeProvider` into stages that use time (alarm debouncing, rolling windows):

```csharp
public sealed class HighTemperatureAlarmStage(
    ContextCache cache,
    IFinalStage next,
    TimeProvider time,
    ILogger<HighTemperatureAlarmStage> logger) : IAlarmStage
{
    public async ValueTask ProcessAsync(CanonicalReading reading, CancellationToken ct)
    {
        var now = time.GetUtcNow();
        // ...
    }
}
```

In production:

```csharp
services.AddSingleton(TimeProvider.System);
```

In tests, use `FakeTimeProvider`:

```csharp
var time = new FakeTimeProvider();
var stage = new HighTemperatureAlarmStage(cache, next, time, logger);

await stage.ProcessAsync(reading, default);
time.Advance(TimeSpan.FromSeconds(5));
await stage.ProcessAsync(reading, default);
```

## What not to test

- `Channel<T>` mechanics.
- `System.IO.Pipelines` mechanics.
- DI resolution (one integration test for the host as a whole is enough).
- Transport libraries' internals.

## What to test hard

- **Frame parsing edge cases:** short reads, split across pipe segments, trailing garbage, malformed length prefix.
- **Alarm state transitions** around restart — does the alarm re-fire if the pipeline restarts mid-condition? Should it?
- **Backpressure and overflow.** Fill the sink channel, assert the drop policy. Fill the serial channel, assert reader-thread behavior.
- **Hot reload.** Change calibration mid-run, assert stage 3 picks up the new value without restart.
- **Config seeding edge cases.** Empty YAML, invalid YAML, YAML referencing unknown sensor types, `--reseed` behavior.

## Scheduler caveat

Tests that use dedicated-thread transports (serial, Bluetooth) need a way to deterministically drive the thread. Two options:

1. **Avoid testing them via the real thread.** Mock `SerialPort` and drive the read callback directly from the test.
2. **Use `Task.Delay` with a short timeout** and `TimeProvider` to wait for the channel to have events. Less deterministic; only use as a last resort.

Prefer option 1. Virtual time beats wall-clock waits.
