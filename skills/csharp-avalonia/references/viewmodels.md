# ViewModels

## Validation

Inherit from `ObservableValidator` instead of `ObservableObject`. Use data annotations or custom `ValidateProperty` calls.

```csharp
public partial class LoginViewModel : ObservableValidator
{
    [ObservableProperty]
    [Required(ErrorMessage = "Username is required")]
    [MinLength(3, ErrorMessage = "At least 3 characters")]
    [NotifyDataErrorInfo]
    public partial string Username { get; set; } = "";

    [ObservableProperty]
    [Required]
    [MinLength(8)]
    [NotifyDataErrorInfo]
    public partial string Password { get; set; } = "";

    [RelayCommand(CanExecute = nameof(CanLogin))]
    private async Task LoginAsync()
    {
        ValidateAllProperties();
        if (HasErrors) return;
        // ...
    }

    private bool CanLogin() => !HasErrors;
}
```

`[NotifyDataErrorInfo]` triggers validation on property change. `ValidateAllProperties()` runs all validators at once (on submit). The Avalonia binding system picks up `INotifyDataErrorInfo` and displays errors via `DataValidationErrors` in the view.

Remember to **remove `DataAnnotationsValidationPlugin`** in `App.axaml.cs` when using `ObservableValidator` — otherwise errors duplicate.

## Messaging (WeakReferenceMessenger)

For VM-to-VM communication that's too loose for direct injection (e.g., a status bar VM listening for errors from any other VM), use the toolkit's messenger.

Message type:

```csharp
public sealed record UserLoggedInMessage(string Username);
```

Publisher:

```csharp
WeakReferenceMessenger.Default.Send(new UserLoggedInMessage(Username));
```

Subscriber:

```csharp
public sealed partial class StatusBarViewModel : ObservableObject, IRecipient<UserLoggedInMessage>
{
    public StatusBarViewModel()
    {
        WeakReferenceMessenger.Default.Register(this);
    }

    public void Receive(UserLoggedInMessage message)
    {
        CurrentUser = message.Username;
    }
}
```

### Rules

- **`WeakReferenceMessenger`, not `StrongReferenceMessenger`.** Weak references let subscribers get GC'd naturally. Strong references require manual unregister and leak if you forget.
- **Messages as `record` or `record struct`.** Immutable, structural equality, `required` members for required fields.
- **Don't overuse.** Messenger is for genuinely decoupled scenarios. If two VMs have a clear parent-child relationship, inject one into the other.

## Navigation

Two workable patterns. Pick one and stick with it.

### Pattern A: Navigation service

```csharp
public interface INavigationService
{
    void NavigateTo<TViewModel>() where TViewModel : ObservableObject;
    void NavigateTo<TViewModel>(object parameter) where TViewModel : ObservableObject;
    void GoBack();
}
```

The service holds a reference to a container VM with a `CurrentViewModel` property. Navigation mutates that property; the view binds to it via a `ContentControl` and `ViewLocator` renders the right view.

Good for single-window apps with swappable content. Standard shell pattern.

### Pattern B: Window-per-operation

For modal dialogs or separate windows, inject an `IDialogService`:

```csharp
public interface IDialogService
{
    Task<TResult?> ShowDialogAsync<TViewModel, TResult>(TViewModel vm)
        where TViewModel : ObservableObject;
}
```

Implementation creates a `Window` hosting the VM, shows it modally, returns the dialog result. Keeps windowing concerns out of VMs.

## Async command patterns

### Cancellation

`[RelayCommand(IncludeCancelCommand = true)]` generates a companion cancel command:

```csharp
[RelayCommand(IncludeCancelCommand = true)]
private async Task LoadAsync(CancellationToken ct)
{
    // ct is cancelled when LoadCancelCommand executes
    await service.DoWorkAsync(ct);
}
```

Bind `LoadCancelCommand` to a Cancel button.

### Re-entrancy

By default, `[RelayCommand]` allows re-entrant invocation. Use `AllowConcurrentExecutions = false` (the default on async methods) — a second invocation is blocked while the first runs. The command's `CanExecute` goes false while executing.

If you *want* concurrent invocations (rare), set `AllowConcurrentExecutions = true`.

### Progress

```csharp
[RelayCommand]
private async Task ExportAsync(CancellationToken ct)
{
    var progress = new Progress<int>(pct => ExportProgress = pct);
    await _exporter.ExportAsync(progress, ct);
}
```

`Progress<T>` reports to the UI thread automatically when created on the UI thread.

## Design-time ViewModels

For the Avalonia XAML designer to render with data, expose a design-time instance:

```csharp
public sealed class DesignMainWindowViewModel : MainWindowViewModel
{
    public DesignMainWindowViewModel()
        : base(new DesignGreetingService())
    {
        Name = "Design-time User";
        Greeting = "Hello, Design-time User";
    }
}
```

In XAML:

```xml
<Design.DataContext>
    <vm:DesignMainWindowViewModel />
</Design.DataContext>
```

Keep design-time VMs out of production DI — they're only for the designer.

## Dispose

ViewModels rarely need `IDisposable`. If a VM holds subscriptions or handles that must be released:

```csharp
public sealed partial class ObservingViewModel : ObservableObject, IDisposable
{
    private readonly IDisposable _subscription;

    public ObservingViewModel(IEventStream stream)
    {
        _subscription = stream.Subscribe(HandleEvent);
    }

    public void Dispose() => _subscription.Dispose();
}
```

Avalonia doesn't dispose DataContext VMs automatically. If the VM is `Transient` and the hosting window closes, the VM becomes unreachable and GC reclaims it — usually fine. Explicit disposal is needed only for non-GC-managed resources.
