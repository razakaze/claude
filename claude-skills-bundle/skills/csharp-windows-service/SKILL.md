---
name: csharp-windows-service
description: Use when writing or modifying a long-running Windows Service in C# 14 / .NET 10, built with the Worker SDK and Microsoft.Extensions.Hosting.WindowsServices. Triggers on mentions of BackgroundService, IHostedService, Worker Service, sc.exe, ServiceBase, Windows Event Log, service lifecycle, graceful shutdown, service install/uninstall, or running a headless .NET process under the Windows Service Control Manager. Use whenever the user is adding a background worker, debugging service startup/shutdown, configuring service recovery, or moving hosted logic between console and service mode.
---

# C# Windows Service (.NET 10 / C# 14)

Build Windows Services as **Worker projects** with one or more `BackgroundService` classes, hosted by `HostApplicationBuilder`, registered via `AddWindowsService()`. Install with `sc.exe`. Log to Windows Event Log in production, console in development. Treat the service as a normal .NET host that happens to be controlled by SCM — not as a special "service-shaped" thing.

---

## Project shape

### csproj

```xml
<Project Sdk="Microsoft.NET.Sdk.Worker">
  <PropertyGroup>
    <TargetFramework>net10.0-windows</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <OutputType>exe</OutputType>
    <RootNamespace>MyCompany.MyService</RootNamespace>
    <RuntimeIdentifier>win-x64</RuntimeIdentifier>
    <PublishSingleFile Condition="'$(Configuration)' == 'Release'">true</PublishSingleFile>
    <SelfContained>true</SelfContained>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.Extensions.Hosting" Version="10.0.*" />
    <PackageReference Include="Microsoft.Extensions.Hosting.WindowsServices" Version="10.0.*" />
  </ItemGroup>
</Project>
```

Common .NET 10 / C# 14 defaults (`Nullable`, `ImplicitUsings`, `TreatWarningsAsErrors`) are explained in [`../_shared/csproj-defaults.md`](../_shared/csproj-defaults.md). This skill overrides `TargetFramework` to `net10.0-windows`.

`net10.0-windows` is required (not plain `net10.0`) — the Windows Services package pulls in Windows-specific APIs. `SelfContained=true` plus `PublishSingleFile=true` produces a single `.exe` you can drop into any Windows box without a framework install.

### Program.cs

```csharp
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Hosting.WindowsServices;

var builder = Host.CreateApplicationBuilder(args);

builder.Services.AddWindowsService(options =>
{
    options.ServiceName = "MyCompany.MyService";
});

builder.Services.AddHostedService<DataPollingWorker>();
builder.Services.AddHostedService<CleanupWorker>();

builder.Services.AddSingleton<IDataSource, DataSource>();
builder.Services.Configure<PollingOptions>(
    builder.Configuration.GetSection("Polling"));

IHost host = builder.Build();
host.Run();
```

`HostApplicationBuilder` is the current idiom. Don't use `Host.CreateDefaultBuilder` — it's legacy. `AddWindowsService` detects whether the process is running under SCM; if not (dev box), it's a no-op and the app runs as a console.

---

## BackgroundService contract

One `BackgroundService` per concern. Not one mega-service with flags. Each subclass owns a single responsibility — naming should reflect that (`DataPollingWorker`, not `MainWorker`).

```csharp
public sealed class DataPollingWorker(
    ILogger<DataPollingWorker> logger,
    IDataSource source,
    IOptions<PollingOptions> options) : BackgroundService
{
    private readonly PollingOptions _options = options.Value;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using PeriodicTimer timer = new(_options.Interval);

        while (await timer.WaitForNextTickAsync(stoppingToken))
        {
            try
            {
                await source.PollAsync(stoppingToken);
            }
            catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested)
            {
                break;
            }
            catch (Exception ex)
            {
                logger.LogError(ex, "Poll iteration failed");
            }
        }
    }
}
```

### Rules

- **`PeriodicTimer`, not `Task.Delay`**, for fixed-interval work. It doesn't drift under load and doesn't allocate per tick.
- **Honor `stoppingToken`.** When SCM calls stop, the host signals the token and waits up to 30 seconds. If `ExecuteAsync` doesn't return in that window, the service is killed hard and Windows logs a timeout error.
- **Catch exceptions inside the loop, not around it.** An uncaught exception propagates out of `ExecuteAsync`, which since .NET 6 defaults to `StopHost` — the entire service crashes. For transient errors (network, DB), log and continue. For non-recoverable errors, let it propagate.
- **Don't swallow `OperationCanceledException` when the token is the stopping token.** Let the loop exit cleanly.
- **`sealed` on the class** — these aren't meant to be inherited. Primary constructors (C# 12+) for DI.

### When to crash on purpose

Exceptions that indicate the service can't do its job at all — missing configuration, database schema mismatch, unreachable required service at startup — should crash the host. Don't trap them. Windows restarts the service per SCM recovery config, and the next start either succeeds or crashes again visibly. Silent degradation is worse than restart loops.

### Scoped services inside a BackgroundService

`BackgroundService` is a singleton. If you need a scoped dependency (typical for EF Core `DbContext`), create a scope per unit of work:

```csharp
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    using PeriodicTimer timer = new(_options.Interval);
    while (await timer.WaitForNextTickAsync(stoppingToken))
    {
        await using var scope = scopeFactory.CreateAsyncScope();
        var repo = scope.ServiceProvider.GetRequiredService<IReportRepository>();
        await repo.ProcessPendingAsync(stoppingToken);
    }
}
```

Inject `IServiceScopeFactory`, not `IServiceProvider`.

See `references/background-service.md` for startup ordering, multiple workers, and `StartAsync`/`StopAsync` overrides.

---

## Logging

Windows Event Log in production, console in development. The host auto-wires Event Log when running as a service (`AddWindowsService` does this), so you don't configure it explicitly — you just configure the **filter level** per sink.

```json
// appsettings.json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.Hosting.Lifetime": "Information"
    },
    "EventLog": {
      "LogLevel": {
        "Default": "Warning"
      },
      "SourceName": "MyCompany.MyService"
    }
  }
}
```

### Rules

- **Event Log is for operators, not developers.** Warnings and errors. Don't write `Information` there — it floods the log and operators ignore the whole source.
- **Structured logging only.** `logger.LogError(ex, "Poll failed for {SensorId}", sensorId)` — never string interpolation. Log processors depend on the message template staying stable.
- **Correlation IDs** for cross-worker traces. Use `ActivitySource` / OpenTelemetry if the service talks to anything external. Don't invent your own ID scheme.
- **No secrets in logs.** Connection strings, tokens, PII — redact or omit.

The Event Log source must exist before the service writes to it the first time. Either create it during install (requires admin), or let the first write create it (requires admin on first run only). The installer path is cleaner.

See `references/logging.md` for OpenTelemetry wiring, log rotation for file sinks, and filtering patterns.

---

## Configuration

Standard `Microsoft.Extensions.Configuration` layering:

1. `appsettings.json` — baseline, committed.
2. `appsettings.{Environment}.json` — per-environment overrides.
3. Environment variables — deployment overrides.
4. Command-line — ad-hoc.

`IOptions<T>` / `IOptionsMonitor<T>` / `IOptionsSnapshot<T>` — pick based on reload needs:

- `IOptions<T>` — read once at DI construction. Doesn't reload. Use for values that can't change at runtime.
- `IOptionsMonitor<T>` — singleton, fires `OnChange` when the config source changes. Use for values that must reload without restart.
- `IOptionsSnapshot<T>` — per-scope. Doesn't apply in a singleton like `BackgroundService`. Don't use here.

For a Windows Service, the config source is usually the JSON file next to the exe. File watchers detect changes and re-fire `IOptionsMonitor`. But **not all settings should hot-reload** — changing a poll interval mid-run is fine, changing a database connection string mid-run is asking for state inconsistencies. Make the hot-reload choice explicit in the `Options` type's XML doc.

```csharp
public sealed class PollingOptions
{
    /// <summary>Hot-reloadable.</summary>
    public TimeSpan Interval { get; set; } = TimeSpan.FromSeconds(30);

    /// <summary>Requires service restart.</summary>
    public string ConnectionString { get; set; } = "";

    public required string ServiceEndpoint { get; set; }
}
```

The `required` keyword (C# 11+) is the right tool for "config must provide this or we don't start" — it makes deserialization fail loudly at startup, not silently at first use.

---

## Install, uninstall, update

**Single-file publish:**

```
dotnet publish -c Release -r win-x64 --self-contained true
```

Output lands in `bin\Release\net10.0-windows\win-x64\publish\`.

**Install as service (admin cmd):**

```cmd
sc.exe create "MyCompany.MyService" ^
  binPath= "C:\Services\MyCompany.MyService\MyCompany.MyService.exe" ^
  start= auto ^
  DisplayName= "My Company Service"

sc.exe description "MyCompany.MyService" "Long description for operators."

sc.exe failure "MyCompany.MyService" reset= 3600 actions= restart/5000/restart/30000/restart/60000

sc.exe start "MyCompany.MyService"
```

**Uninstall:**

```cmd
sc.exe stop "MyCompany.MyService"
sc.exe delete "MyCompany.MyService"
```

**Update:** stop the service, replace the exe, start. The exe is locked while the service runs. Scripting this as a `.cmd` or PowerShell script in the repo is worth the five minutes — operators will use it and you won't get pinged.

### Running as which account

Default is `LocalSystem` — broad permissions, can't access network shares with user credentials. For most services this is wrong. Specify a dedicated service account:

```cmd
sc.exe create "MyCompany.MyService" binPath= "..." obj= "NT AUTHORITY\NetworkService"
```

Or a domain account with a managed password (gMSA). Never a personal user account — those get password-rotated and the service breaks.

See `references/deployment.md` for MSI via WiX, Advanced Installer, and PowerShell install scripts.

---

## Debugging

### Dev loop

Run the project normally from Visual Studio 2026 or `dotnet run`. `AddWindowsService` detects non-SCM context and the host runs as a console. Ctrl+C triggers `stoppingToken` — same graceful shutdown path as SCM stop. This is the fast iteration path; use it for 95% of development.

### Debugging as an installed service

When a bug only reproduces under SCM (different account, different working directory, different env vars):

1. Install the service, start it.
2. Visual Studio → Debug → Attach to Process → find your exe → attach.
3. Breakpoints now hit on the running service.

Don't try to start a service under the debugger directly — SCM has a 30-second start timeout that debugger pauses will blow through.

For startup bugs specifically (crash before you can attach):
- Add a `Debugger.Launch()` at the top of `Program.cs` guarded by a config flag or env var.
- Install the service, start it — Windows prompts you to attach a debugger.
- Remove before shipping.

### Common startup failures

- **Service starts then stops immediately.** Unhandled exception in `ExecuteAsync` before the first `await`. Check Event Viewer → Windows Logs → Application.
- **Service fails to start, error 1053 "did not respond in timely fashion".** Long synchronous work in `StartAsync` or `ExecuteAsync` before first `await`. Move it off the startup path or run it inside `ExecuteAsync` after yielding.
- **Access denied on file/registry/network.** Service account lacks the permission. Don't fix by running as `LocalSystem` in production — grant the specific permission to the dedicated account.

---

## Testing

`BackgroundService` classes are normal classes. Test the logic, not the hosting.

```csharp
[Fact]
public async Task PollingWorker_LogsErrorAndContinues_WhenSourceThrows()
{
    var source = Substitute.For<IDataSource>();
    source.PollAsync(Arg.Any<CancellationToken>()).Returns(
        Task.FromException(new HttpRequestException("transient")),
        Task.CompletedTask);

    var logger = Substitute.For<ILogger<DataPollingWorker>>();
    var options = Options.Create(new PollingOptions { Interval = TimeSpan.FromMilliseconds(10) });
    var worker = new DataPollingWorker(logger, source, options);

    using var cts = new CancellationTokenSource(TimeSpan.FromMilliseconds(100));
    await worker.StartAsync(cts.Token);
    await Task.Delay(50);
    await worker.StopAsync(CancellationToken.None);

    await source.Received(2).PollAsync(Arg.Any<CancellationToken>());
}
```

Don't spin up a real `Host`, don't use `TestServer` (that's for ASP.NET Core). Call `StartAsync`/`StopAsync` directly.

For things that *do* need the host (DI wiring, config loading, startup ordering), a single integration test with `HostApplicationBuilder` building a real host and `Host.StartAsync` → assert → `Host.StopAsync` is enough. Don't write one of those per worker.

See `references/testing.md` for `PeriodicTimer` in tests (use `TimeProvider.Fake`), logging assertions, and integration harness patterns.

---

## C# 14 idioms worth using here

- **Primary constructors** for `BackgroundService` classes — DI boilerplate shrinks by half.
- **`field` keyword** for properties with light validation logic — no more `_backingField` noise.
- **Collection expressions** (`[]`) for small fixed collections in options/config defaults.
- **`required` members** on `Options` types to force valid config at startup.
- **Null-conditional assignment** (`x?.Property = value`) where it clarifies. Don't reach for it just because it's new.

Extension members (the headline C# 14 feature) are great for domain types but rarely the right tool inside a `BackgroundService`. Resist adding extensions on `CancellationToken` or `ILogger` — the base APIs are fine; extensions mostly add indirection.

---

## What this skill does not cover

- Hosting ASP.NET Core inside the Windows Service (`UseWindowsService` on `WebApplicationBuilder`). Possible and supported, but a separate pattern — the REST service skill covers that.
- Database access from a worker. Use EF Core via scoped services; see the EF Core skill.
- GUI integration (service + tray app). Different architecture — use named pipes or localhost HTTP, don't try to share a process.
- Non-Windows hosts. This service targets Windows only. For cross-platform daemons, use `UseSystemd()` on Linux in a separate build configuration.
