# Pipeline Construction

## Stage interfaces

Each stage is an interface with one primary method. Construction via primary-constructor DI.

```csharp
public interface IIntegrationStage
{
    Task StartAsync(CancellationToken ct);
    Task StopAsync(CancellationToken ct);
}

public interface IProtocolStage
{
    ValueTask ProcessAsync(ReadOnlyMemory<byte> frame, CancellationToken ct);
}

public interface INormalizationStage
{
    ValueTask ProcessAsync(RawReading reading, CancellationToken ct);
}

public interface IAlarmStage
{
    ValueTask ProcessAsync(CanonicalReading reading, CancellationToken ct);
}

public interface IFinalStage
{
    ValueTask ProcessAsync(SensorEvent evt, CancellationToken ct);
}
```

Domain types (`RawReading`, `CanonicalReading`, `SensorEvent`) are sensor-family specific. Define as `readonly record struct` on the hot path.

## Registries

Resolve a stage implementation by sensor type or protocol name. One registry per stage kind.

```csharp
public interface IProtocolStageFactory
{
    string Protocol { get; }
    IProtocolStage Create(INormalizationStage next, ILoggerFactory loggerFactory);
}

public sealed class ProtocolStageRegistry(IEnumerable<IProtocolStageFactory> factories)
{
    private readonly Dictionary<string, IProtocolStageFactory> _map =
        factories.ToDictionary(f => f.Protocol, StringComparer.OrdinalIgnoreCase);

    public IProtocolStage Resolve(string protocol, INormalizationStage next, ILoggerFactory logs)
        => _map.TryGetValue(protocol, out var f)
            ? f.Create(next, logs)
            : throw new InvalidOperationException($"Unknown protocol: {protocol}");
}
```

Adding a new protocol: implement `IProtocolStageFactory`, register with DI. Done. No edits to `PipelineFactory`, no edits to the registry.

```csharp
public sealed class TemperatureProtocolStageFactory : IProtocolStageFactory
{
    public string Protocol => "temperature-v1";

    public IProtocolStage Create(INormalizationStage next, ILoggerFactory logs) =>
        new TemperatureProtocolStage(next, logs.CreateLogger<TemperatureProtocolStage>());
}

services.AddSingleton<IProtocolStageFactory, TemperatureProtocolStageFactory>();
```

Same pattern for `IIntegrationStageFactory`, `INormalizationStageFactory`, `IAlarmStageFactory`.

## Factory

```csharp
public sealed class PipelineFactory(
    IEnumerable<IIntegrationStageFactory> integrationFactories,
    ProtocolStageRegistry protocols,
    NormalizationStageRegistry normalizers,
    AlarmStageRegistry alarms,
    ISensorEventSink finalSink,
    ISensorConfigRepository repo,
    ILoggerFactory loggerFactory)
{
    public PipelineInstance Build(string sensorId)
    {
        var config = repo.LoadEnabled(sensorId)
            ?? throw new InvalidOperationException($"no config: {sensorId}");

        var cache = new ContextCache(sensorId);
        LoadCalibrationInto(cache, config);
        LoadAlarmRulesInto(cache, config);

        var finalStage = new FinalStage(finalSink, config,
            loggerFactory.CreateLogger<FinalStage>());
        var alarmStage = alarms.Resolve(config.SensorType, cache, finalStage, loggerFactory);
        var normStage  = normalizers.Resolve(config.SensorType, cache, alarmStage, loggerFactory);
        var protoStage = protocols.Resolve(config.Protocol, normStage, loggerFactory);

        var integrationFactory = integrationFactories.FirstOrDefault(f => f.Supports(config.Transport))
            ?? throw new InvalidOperationException(
                $"No integration for transport: {config.Transport.Type}");

        var integration = integrationFactory.Create(config, protoStage, loggerFactory);

        return new PipelineInstance(sensorId, integration, cache, loggerFactory.CreateLogger<PipelineInstance>());
    }

    private void LoadCalibrationInto(ContextCache cache, SensorConfig config)
    {
        foreach (var (key, value) in config.Calibration)
            cache.Put($"calibration.{key}", value);
    }

    private void LoadAlarmRulesInto(ContextCache cache, SensorConfig config)
    {
        cache.Put("alarm.rules", config.AlarmRules);
    }
}
```

Stages are wired **end-to-start** — the final stage first, then backwards — because each stage needs a reference to its downstream neighbor.

## PipelineInstance

```csharp
public sealed class PipelineInstance(
    string sensorId,
    IIntegrationStage integration,
    ContextCache cache,
    ILogger<PipelineInstance> logger)
{
    public string SensorId { get; } = sensorId;
    public ContextCache Cache { get; } = cache;

    public async Task StartAsync(CancellationToken ct)
    {
        logger.LogInformation("Pipeline {Id} starting", SensorId);
        await integration.StartAsync(ct);
    }

    public async Task StopAsync(CancellationToken ct)
    {
        logger.LogInformation("Pipeline {Id} stopping", SensorId);
        try
        {
            await integration.StopAsync(ct);
        }
        catch (Exception ex)
        {
            logger.LogWarning(ex, "Pipeline {Id} stop threw", SensorId);
        }
    }
}
```

`PipelineInstance` owns the integration stage and the context cache. Downstream stages are held by reference chains originating at the integration stage — they stay alive as long as the integration does.

## PipelineManager

The `IHostedService` that owns all pipeline instances and responds to runtime config changes.

```csharp
public sealed class PipelineManager(
    PipelineFactory factory,
    ISensorConfigRepository repo,
    ILogger<PipelineManager> logger) : IHostedService
{
    private readonly ConcurrentDictionary<string, PipelineInstance> _running = new();

    public async Task StartAsync(CancellationToken ct)
    {
        foreach (var id in repo.AllEnabledSensorIds())
        {
            await StartOneAsync(id, ct);
        }
    }

    public async Task StartOneAsync(string sensorId, CancellationToken ct)
    {
        if (_running.ContainsKey(sensorId))
        {
            logger.LogWarning("Pipeline {Id} already running", sensorId);
            return;
        }
        try
        {
            var instance = factory.Build(sensorId);
            await instance.StartAsync(ct);
            _running[sensorId] = instance;
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "Failed to start pipeline {Id}", sensorId);
            throw;
        }
    }

    public async Task StopOneAsync(string sensorId, CancellationToken ct)
    {
        if (_running.TryRemove(sensorId, out var instance))
        {
            await instance.StopAsync(ct);
        }
    }

    public async Task RestartOneAsync(string sensorId, CancellationToken ct)
    {
        await StopOneAsync(sensorId, ct);
        await StartOneAsync(sensorId, ct);
    }

    public void HotReloadCalibration(string sensorId, IReadOnlyDictionary<string, object> calibration)
    {
        if (!_running.TryGetValue(sensorId, out var instance)) return;
        foreach (var (key, value) in calibration)
            instance.Cache.Put($"calibration.{key}", value);
    }

    public void HotReloadAlarmRules(string sensorId, IReadOnlyList<AlarmRule> rules)
    {
        if (!_running.TryGetValue(sensorId, out var instance)) return;
        instance.Cache.Put("alarm.rules", rules);
    }

    public async Task StopAsync(CancellationToken ct)
    {
        await Task.WhenAll(_running.Values.Select(i => i.StopAsync(ct)));
        _running.Clear();
    }
}
```

Transport and protocol changes go through `RestartOneAsync`. Calibration and alarm rule changes go through the `HotReload*` methods — no restart.

## Outbound sink

The final-stage sink is the pipeline → Orchestration Manager boundary. Two shapes per the main SKILL.md rule (direct call vs channel).

### Direct call

```csharp
public sealed class DirectSensorEventSink(IOrchestrationManager orchestration) : ISensorEventSink
{
    public ValueTask PublishAsync(SensorEvent evt, CancellationToken ct)
        => orchestration.HandleAsync(evt, ct);
}
```

The Orchestration Manager implements `IOrchestrationManager.HandleAsync` and does whatever it does (persist, broadcast, etc).

### Channel

```csharp
public sealed class ChannelSensorEventSink : ISensorEventSink, IHostedService
{
    private readonly Channel<SensorEvent> _channel =
        Channel.CreateBounded<SensorEvent>(new BoundedChannelOptions(1024)
        {
            FullMode = BoundedChannelFullMode.DropOldest,
            SingleReader = true,
            SingleWriter = false,
        });

    public ChannelReader<SensorEvent> Reader => _channel.Reader;

    public ValueTask PublishAsync(SensorEvent evt, CancellationToken ct)
    {
        _channel.Writer.TryWrite(evt);  // drop on overflow by policy
        return ValueTask.CompletedTask;
    }

    public Task StartAsync(CancellationToken ct) => Task.CompletedTask;

    public Task StopAsync(CancellationToken ct)
    {
        _channel.Writer.TryComplete();
        return Task.CompletedTask;
    }
}
```

The Orchestration Manager consumes from `Reader` in its own `IHostedService`:

```csharp
public sealed class OrchestrationManager(
    ChannelSensorEventSink source,
    /* DB, HTTP, whatever */) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await foreach (var evt in source.Reader.ReadAllAsync(stoppingToken))
        {
            await HandleAsync(evt, stoppingToken);
        }
    }
}
```

Register whichever sink the project uses:

```csharp
// Default: direct
services.AddSingleton<ISensorEventSink, DirectSensorEventSink>();

// Or: channel (and the Orchestration Manager subscribes)
services.AddSingleton<ChannelSensorEventSink>();
services.AddSingleton<ISensorEventSink>(sp => sp.GetRequiredService<ChannelSensorEventSink>());
services.AddHostedService(sp => sp.GetRequiredService<ChannelSensorEventSink>());
services.AddHostedService<OrchestrationManager>();
```

## Wiring everything in Program.cs

```csharp
var builder = WebApplication.CreateBuilder(args);

// SQLite
var connString = builder.Configuration.GetConnectionString("Default")!;
builder.Services.AddSingleton<ISensorConfigRepository>(_ => new SensorConfigRepository(connString));

// Stage factories
builder.Services.AddSingleton<IProtocolStageFactory, TemperatureProtocolStageFactory>();
builder.Services.AddSingleton<IProtocolStageFactory, PressureProtocolStageFactory>();
// ... etc

builder.Services.AddSingleton<ProtocolStageRegistry>();
builder.Services.AddSingleton<NormalizationStageRegistry>();
builder.Services.AddSingleton<AlarmStageRegistry>();

builder.Services.AddSingleton<IIntegrationStageFactory, UdpIntegrationFactory>();
builder.Services.AddSingleton<IIntegrationStageFactory, TcpIntegrationFactory>();
builder.Services.AddSingleton<IIntegrationStageFactory, SerialIntegrationFactory>();

// Sink (pick one)
builder.Services.AddSingleton<ISensorEventSink, DirectSensorEventSink>();

// Factory and manager
builder.Services.AddSingleton<PipelineFactory>();
builder.Services.AddHostedService<PipelineManager>();

// Orchestration Manager's REST API
builder.Services.AddAuthentication(...).AddJwtBearer(...);
builder.Services.AddAuthorization();
// ... endpoints per csharp-rest-aspnetcore

// Kestrel for TCP sensor ports + HTTP
builder.WebHost.ConfigureKestrel(k =>
{
    k.ListenAnyIP(5000);                                 // HTTP
    k.ListenAnyIP(9000, lo => lo.UseConnectionHandler<TcpIntegrationStage>());  // sensor TCP
});

// Seed YAML → SQLite on startup (once, before anything else reads config)
var seeder = new YamlSensorSeeder(...);
seeder.Seed("sensors.yaml", connString, reseed: args.Contains("--reseed"));

var app = builder.Build();
app.UseAuthentication();
app.UseAuthorization();
app.MapSensorAdminApi();   // hot-reload endpoints
app.MapHealthChecks("/health");

await app.RunAsync();
```

## Error handling at the pipeline boundary

Stage exceptions must not kill the pipeline. The integration stage catches, logs, and continues.

```csharp
while (!ct.IsCancellationRequested)
{
    try
    {
        await next.ProcessAsync(frame, ct);
    }
    catch (OperationCanceledException) { throw; }
    catch (Exception ex)
    {
        logger.LogWarning(ex, "Pipeline frame processing failed for {Id}", config.SensorId);
        // metric: pipeline_errors_total
    }
}
```

Only `OperationCanceledException` propagates. Everything else logs and drops the frame.

Exceptions that indicate fundamental misconfiguration (missing calibration key, unknown sensor type) should be caught in the **factory** and turned into startup failures — never reach runtime.
