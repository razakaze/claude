# DI Patterns

## Service lifetimes

| Lifetime | When |
|---|---|
| Singleton | Stateless services, caches, repositories with internal pooling |
| Transient | ViewModels, per-use services |
| Scoped | Rarely useful in desktop apps — no natural scope boundary |

Desktop apps don't have an HTTP-request-shaped scope. If you need scoped-like behavior (e.g., an EF Core `DbContext` per unit of work), use `IServiceScopeFactory` and create scopes explicitly around the unit of work — usually inside a service method, not a VM.

```csharp
public sealed class DataImporter(IServiceScopeFactory scopes)
{
    public async Task ImportAsync(CancellationToken ct)
    {
        await using var scope = scopes.CreateAsyncScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        // ...
    }
}
```

## HostApplicationBuilder alternative

Instead of hand-rolling `ServiceCollection`, use `HostApplicationBuilder` for access to configuration, logging, and hosted services:

```csharp
public static IServiceProvider BuildHost()
{
    var builder = Host.CreateApplicationBuilder();

    builder.Services.AddSingleton<IGreetingService, GreetingService>();
    builder.Services.AddTransient<MainWindowViewModel>();
    builder.Services.AddTransient<MainWindow>();

    builder.Services.AddHostedService<BackgroundSyncWorker>();

    var host = builder.Build();
    _ = host.StartAsync();  // fire and forget hosted services
    return host.Services;
}
```

The `AddHostedService` route lets you run `BackgroundService`s inside a desktop app — useful for polling, sync, or any continuous work. Register them, they start when the host starts, stop when the app exits.

On app shutdown, call `host.StopAsync()` from the desktop lifetime's `Exit` event to let hosted services shut down gracefully.

## Logging

`HostApplicationBuilder` wires logging automatically. Configure through `appsettings.json`:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning"
    }
  }
}
```

Inject `ILogger<T>` into any service or VM. Avoid `ILoggerFactory` directly.

## Factory pattern for dynamic VMs

When you need to create a VM at runtime with a parameter (e.g., detail view for an entity), use a factory delegate rather than injecting `IServiceProvider` everywhere:

```csharp
public delegate DetailViewModel DetailViewModelFactory(Guid itemId);

services.AddTransient<DetailViewModel>();
services.AddSingleton<DetailViewModelFactory>(sp =>
    itemId => ActivatorUtilities.CreateInstance<DetailViewModel>(sp, itemId));
```

Then inject `DetailViewModelFactory` where you need to create detail VMs:

```csharp
public partial class MainWindowViewModel(DetailViewModelFactory detailFactory) : ObservableObject
{
    [RelayCommand]
    private void OpenDetail(Guid id)
    {
        var vm = detailFactory(id);
        // navigate to detail...
    }
}
```

`ActivatorUtilities.CreateInstance<T>` resolves the rest of the constructor params from DI and fills the remaining ones from the arguments you pass.

## Anti-patterns

### Service locator

```csharp
// BAD
public partial class BadViewModel : ObservableObject
{
    private readonly IGreetingService _svc = App.Current.Services.GetRequiredService<IGreetingService>();
}
```

Hidden dependencies, untestable, resolves at construction time with no error visibility. Always inject via constructor.

### Singleton ViewModels

```csharp
// BAD
services.AddSingleton<MainWindowViewModel>();
```

State from a closed window survives into the next opening. Commands wired to dead UI elements. Only make a VM singleton if it genuinely represents app-wide state (rare — and even then, usually a *service* is the right home for that state, not a VM).

### Capturing the root provider

```csharp
// BAD
public App()
{
    _provider = ConfigureServices();
    ServiceLocator.Current = _provider;  // static, global, forever
}
```

Global state. Breaks scope semantics. Makes testing impossible.
