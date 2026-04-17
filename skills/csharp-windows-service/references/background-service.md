# BackgroundService Patterns

## Startup ordering

Hosted services start **sequentially** in registration order. `StartAsync` on service N blocks service N+1. If `StartAsync` on one service does long synchronous work, it delays every later service.

**Rule:** `StartAsync` does nothing but register state. Long work lives in `ExecuteAsync`, which is called after `StartAsync` returns.

`BackgroundService`'s default `StartAsync` kicks off `ExecuteAsync` on the thread pool and returns immediately — this is correct, don't override it unless you have a specific reason.

## Multiple workers

Register each as `AddHostedService<T>()`. They run in parallel once all `StartAsync`s complete.

```csharp
builder.Services.AddHostedService<DataPollingWorker>();
builder.Services.AddHostedService<CleanupWorker>();
builder.Services.AddHostedService<HealthReporterWorker>();
```

They share the host's lifetime and the same `stoppingToken`. When SCM stops the service, all workers get cancelled in parallel.

**Inter-worker communication:** use DI-registered singletons (`Channel<T>` for producer/consumer, `IMemoryCache` for lookups). Don't reference one worker from another directly — that couples lifecycles in ways that bite on shutdown.

```csharp
builder.Services.AddSingleton(Channel.CreateBounded<WorkItem>(
    new BoundedChannelOptions(1024)
    {
        FullMode = BoundedChannelFullMode.Wait,
        SingleReader = true,
        SingleWriter = false,
    }));
```

Producer worker writes to `channel.Writer`, consumer worker reads from `channel.Reader`. Clean separation, testable independently.

## StartAsync and StopAsync overrides

Override only when you need setup/teardown that can't be expressed inside `ExecuteAsync`.

```csharp
public sealed class FileWatcherWorker(ILogger<FileWatcherWorker> logger) : BackgroundService
{
    private FileSystemWatcher? _watcher;

    public override async Task StartAsync(CancellationToken cancellationToken)
    {
        _watcher = new FileSystemWatcher(@"C:\Incoming");
        _watcher.Created += OnCreated;
        _watcher.EnableRaisingEvents = true;
        await base.StartAsync(cancellationToken);
    }

    public override async Task StopAsync(CancellationToken cancellationToken)
    {
        if (_watcher is not null)
        {
            _watcher.EnableRaisingEvents = false;
            _watcher.Dispose();
        }
        await base.StopAsync(cancellationToken);
    }

    protected override Task ExecuteAsync(CancellationToken stoppingToken) =>
        Task.Delay(Timeout.Infinite, stoppingToken);  // idle, work happens in events
}
```

Always call `base.StartAsync` and `base.StopAsync`.

## Graceful shutdown

`stoppingToken` fires when SCM requests stop. The host then waits up to `ShutdownTimeout` (default 30s) for `ExecuteAsync` to complete.

**Configure the timeout explicitly** if your workers need longer:

```csharp
builder.Services.Configure<HostOptions>(opts =>
{
    opts.ShutdownTimeout = TimeSpan.FromSeconds(60);
});
```

Don't extend beyond 60s — Windows will start showing "not responding" dialogs to operators.

**Inside workers,** propagate the token to every `await`:

```csharp
await dbContext.SaveChangesAsync(stoppingToken);
await httpClient.PostAsync(url, content, stoppingToken);
```

A worker that ignores the token is a worker that blocks shutdown.

## Exception behavior

Default since .NET 6 is `BackgroundServiceExceptionBehavior.StopHost` — an uncaught exception in any `BackgroundService` crashes the whole host.

**This is the right default.** Keep it. Zombies (the old default `Ignore`) are worse than crashes.

Override only for extreme cases where one worker's death genuinely shouldn't affect others:

```csharp
builder.Services.Configure<HostOptions>(opts =>
{
    opts.BackgroundServiceExceptionBehavior = BackgroundServiceExceptionBehavior.Ignore;
});
```

If you do this, add alerting. A silently-dead worker in a running service is a nightmare to diagnose.

## IHostedService vs BackgroundService

`BackgroundService` is the 95% case. Use `IHostedService` directly only when:

- The work is purely event-driven (no loop) and lives entirely in `StartAsync`/`StopAsync`.
- You need to control the `Task` returned from `StartAsync` explicitly (rare).

For the 95% case, `BackgroundService` is less code and handles the `Task` lifecycle correctly.

## Async lifetime anti-pattern

Don't do this:

```csharp
// BAD
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    await _longInitialization;  // blocks the host start path
    while (!stoppingToken.IsCancellationRequested)
    {
        // ...
    }
}
```

`ExecuteAsync` runs before the host reports "started" to SCM. If the `await` is long, Windows times out and kills the service.

Fix: yield immediately, then do init inside the loop.

```csharp
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    await Task.Yield();  // release the start path
    await InitializeAsync(stoppingToken);
    while (!stoppingToken.IsCancellationRequested)
    {
        // ...
    }
}
```

The host considers `ExecuteAsync` "started" as soon as it hits its first real async yield.
