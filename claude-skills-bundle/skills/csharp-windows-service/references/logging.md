# Logging

## Sinks

| Sink | When | Why |
|---|---|---|
| Console | Development | Instant feedback, no setup |
| Windows Event Log | Production, always | Operators watch Event Viewer, ops tools tail it |
| File (Serilog/NLog) | Production, supplementary | Debugging, keeps more detail than Event Log |
| OpenTelemetry OTLP | Production, when you have a collector | Distributed tracing, metrics |

Default stack for a standalone Windows Service: Console (dev) + Event Log (prod) + File (prod). Add OTLP when there's actually a collector to send to — don't add the dependency speculatively.

## Event Log source

The source name is the string operators will filter on in Event Viewer. Use the service name:

```json
{
  "Logging": {
    "EventLog": {
      "SourceName": "MyCompany.MyService",
      "LogName": "Application",
      "LogLevel": { "Default": "Warning" }
    }
  }
}
```

The source must exist in the registry before the first write. Three ways to ensure that:

1. **Installer creates it** (`EventLog.CreateEventSource`). Cleanest.
2. **First run as admin creates it** implicitly. Works, but couples first run to install privilege.
3. **Pre-created manually** per server. Don't do this.

## Filter levels

Event Log at `Warning`, console and file at `Information`, Microsoft.* framework namespaces at `Warning` to cut noise:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Information"
    },
    "EventLog": { "LogLevel": { "Default": "Warning" } },
    "Console":  { "LogLevel": { "Default": "Information" } }
  }
}
```

`Microsoft.Hosting.Lifetime` stays at Information because its messages ("Application started", "Application is shutting down") are genuinely useful to keep visible.

## Structured logging

Always use message templates with named parameters. Never `$"..."` interpolation inside a log call.

```csharp
// GOOD
logger.LogInformation("Processed {Count} records for {CustomerId}", count, customerId);

// BAD
logger.LogInformation($"Processed {count} records for {customerId}");
```

Structured logging preserves the parameter values as separate fields. Log processors (Seq, Splunk, Azure Monitor, OpenTelemetry) filter and aggregate on those fields. Interpolation destroys that.

## Scopes

Use scopes to add context that spans multiple log statements:

```csharp
using (logger.BeginScope("CorrelationId={CorrelationId}", Guid.NewGuid()))
{
    logger.LogInformation("Starting work");
    await DoWork();
    logger.LogInformation("Work complete");
}
```

Both log entries carry `CorrelationId`. The console sink ignores scopes by default — enable `IncludeScopes=true` if you want them visible there. Event Log and most structured sinks pick them up automatically.

## Exception logging

Always pass the exception as the first arg:

```csharp
logger.LogError(ex, "Failed to poll {SensorId}", sensorId);
```

Never `logger.LogError("Failed: {Error}", ex.Message)` — that loses the stack trace. Never `logger.LogError(ex.ToString())` — that puts the trace into the message template, which breaks structured sinks.

## OpenTelemetry

When you add tracing, use the standard OTel packages. Don't roll correlation IDs by hand.

```xml
<PackageReference Include="OpenTelemetry.Extensions.Hosting" Version="1.*" />
<PackageReference Include="OpenTelemetry.Exporter.OpenTelemetryProtocol" Version="1.*" />
<PackageReference Include="OpenTelemetry.Instrumentation.Http" Version="1.*" />
```

```csharp
builder.Services.AddOpenTelemetry()
    .ConfigureResource(r => r.AddService("MyCompany.MyService"))
    .WithTracing(t => t
        .AddSource("MyCompany.MyService")
        .AddHttpClientInstrumentation()
        .AddOtlpExporter())
    .WithMetrics(m => m
        .AddMeter("MyCompany.MyService")
        .AddOtlpExporter());
```

Create a static `ActivitySource` per worker:

```csharp
public sealed class DataPollingWorker : BackgroundService
{
    private static readonly ActivitySource ActivitySource = new("MyCompany.MyService");

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var activity = ActivitySource.StartActivity("PollIteration");
        // ...
    }
}
```

The activity ID propagates through `HttpClient` and EF Core calls automatically when their instrumentation is enabled.

## What not to log

- Passwords, tokens, connection strings — ever.
- PII unless the compliance story is clear.
- Large objects serialized for "convenience." Log an ID, not a 50KB blob.
- Every iteration of a tight loop. Log the aggregate.
