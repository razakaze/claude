---
name: csharp-avalonia
description: Use when writing or modifying Avalonia 11 desktop applications in C# 14 / .NET 10, using CommunityToolkit.Mvvm for MVVM and Microsoft.Extensions.DependencyInjection for DI. Triggers on mentions of Avalonia, AXAML, ViewLocator, ObservableObject, ObservableProperty, RelayCommand, IClassicDesktopStyleApplicationLifetime, Window/UserControl, data binding, Dispatcher.UIThread, or desktop GUI work in .NET. Use whenever the user is adding a view, wiring a ViewModel, configuring the app's DI container, handling cross-platform desktop behavior (Windows/macOS/Linux), or testing ViewModels without the UI thread. If the project uses ReactiveUI instead of CommunityToolkit.Mvvm, this skill does not apply.
---

# C# Avalonia (Avalonia 11 / C# 14 / .NET 10)

Build Avalonia apps with **CommunityToolkit.Mvvm** for MVVM and **Microsoft.Extensions.DependencyInjection** for DI. Views are AXAML, ViewModels are C# partial classes with `[ObservableProperty]` partial properties and `[RelayCommand]` methods. No code-behind beyond what the framework forces. No static service locator. ReactiveUI is a valid choice but not this skill's default — if your project already uses ReactiveUI, don't mix.

---

## Project shape

```
MyApp/
├── App.axaml / App.axaml.cs        # app lifecycle, DI container, ViewLocator
├── Program.cs                       # entry point, AppBuilder
├── ViewLocator.cs                   # resolves ViewModel → View
├── Views/
│   ├── MainWindow.axaml / .cs      # code-behind minimal
│   └── Controls/                   # UserControls
├── ViewModels/
│   ├── ViewModelBase.cs            # inherits ObservableObject
│   └── MainWindowViewModel.cs
├── Services/                        # business logic, no UI refs
└── Models/                          # POCOs, records
```

### csproj

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>WinExe</OutputType>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <BuiltInComInteropSupport>true</BuiltInComInteropSupport>
    <ApplicationManifest>app.manifest</ApplicationManifest>
    <AvaloniaUseCompiledBindingsByDefault>true</AvaloniaUseCompiledBindingsByDefault>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Avalonia" Version="11.3.*" />
    <PackageReference Include="Avalonia.Desktop" Version="11.3.*" />
    <PackageReference Include="Avalonia.Themes.Fluent" Version="11.3.*" />
    <PackageReference Include="Avalonia.Fonts.Inter" Version="11.3.*" />
    <PackageReference Include="Avalonia.Diagnostics" Version="11.3.*" Condition="'$(Configuration)' == 'Debug'" />
    <PackageReference Include="CommunityToolkit.Mvvm" Version="8.4.*" />
    <PackageReference Include="Microsoft.Extensions.DependencyInjection" Version="10.0.*" />
    <PackageReference Include="Microsoft.Extensions.Hosting" Version="10.0.*" />
  </ItemGroup>
</Project>
```

Common .NET 10 / C# 14 defaults (`TargetFramework`, `Nullable`, `ImplicitUsings`, `TreatWarningsAsErrors`) are explained in [`../_shared/csproj-defaults.md`](../_shared/csproj-defaults.md).

`AvaloniaUseCompiledBindingsByDefault=true` is non-negotiable. Runtime bindings are ~10× slower and fail silently. Compiled bindings catch errors at build time.

`TargetFramework=net10.0` (not `net10.0-windows`) — Avalonia is cross-platform. Only target `-windows` if you actually use Windows-specific APIs.

### Program.cs

```csharp
internal sealed class Program
{
    [STAThread]
    public static void Main(string[] args) =>
        BuildAvaloniaApp().StartWithClassicDesktopLifetime(args);

    public static AppBuilder BuildAvaloniaApp() =>
        AppBuilder.Configure<App>()
            .UsePlatformDetect()
            .WithInterFont()
            .LogToTrace();
}
```

`StartWithClassicDesktopLifetime` for desktop. `StartLinuxFbDev` / `StartWithBrowserAppLifetime` for other targets — rare, skill scope is desktop.

---

## DI setup in App.axaml.cs

```csharp
public sealed partial class App : Application
{
    public new static App Current => (App)Application.Current!;
    public IServiceProvider Services { get; private set; } = null!;

    public override void Initialize() => AvaloniaXamlLoader.Load(this);

    public override void OnFrameworkInitializationCompleted()
    {
        Services = ConfigureServices();
        DisableAvaloniaDataAnnotationValidation();

        if (ApplicationLifetime is IClassicDesktopStyleApplicationLifetime desktop)
        {
            var mainWindow = Services.GetRequiredService<MainWindow>();
            mainWindow.DataContext = Services.GetRequiredService<MainWindowViewModel>();
            desktop.MainWindow = mainWindow;
        }

        base.OnFrameworkInitializationCompleted();
    }

    private static IServiceProvider ConfigureServices()
    {
        var services = new ServiceCollection();

        services.AddSingleton<IGreetingService, GreetingService>();
        services.AddSingleton<IDialogService, DialogService>();

        services.AddTransient<MainWindowViewModel>();
        services.AddTransient<SettingsViewModel>();

        services.AddTransient<MainWindow>();

        return services.BuildServiceProvider();
    }

    private static void DisableAvaloniaDataAnnotationValidation()
    {
        var toRemove = BindingPlugins.DataValidators
            .OfType<DataAnnotationsValidationPlugin>()
            .ToArray();
        foreach (var plugin in toRemove) BindingPlugins.DataValidators.Remove(plugin);
    }
}
```

### Why this shape

- **`App.Current.Services`** exposed for rare cases where constructor injection isn't available (ViewLocator is the main one). Never use it from ViewModels — inject what you need.
- **Remove `DataAnnotationsValidationPlugin`** when using `ObservableValidator` (from CommunityToolkit). Otherwise validation fires twice and duplicate errors appear in the UI. This is a known Avalonia + Toolkit interaction.
- **ViewModels as `Transient`.** A new VM per navigation/window instance. Singletons are almost always wrong for VMs — they leak state across screens.
- **Services as `Singleton`** unless they carry per-user or per-session state, in which case scope carefully.

See `references/di-patterns.md` for scoped services, `IHostedService` integration, and `HostApplicationBuilder` as an alternative to hand-rolled `ServiceCollection`.

---

## ViewModels

Inherit from `ObservableObject` (or `ObservableValidator` if you use validation). Use partial properties and `[RelayCommand]`.

```csharp
public partial class MainWindowViewModel(IGreetingService greetings) : ObservableObject
{
    [ObservableProperty]
    public partial string Name { get; set; } = "";

    [ObservableProperty]
    [NotifyCanExecuteChangedFor(nameof(GreetCommand))]
    public partial bool IsBusy { get; set; }

    [ObservableProperty]
    public partial string Greeting { get; set; } = "";

    [RelayCommand(CanExecute = nameof(CanGreet))]
    private async Task GreetAsync(CancellationToken ct)
    {
        IsBusy = true;
        try
        {
            Greeting = await greetings.GetGreetingAsync(Name, ct);
        }
        finally
        {
            IsBusy = false;
        }
    }

    private bool CanGreet() => !IsBusy && !string.IsNullOrWhiteSpace(Name);
}
```

### Rules

- **Partial properties, not fields with attributes.** Since Toolkit 8.4 + C# 13+, the idiom is `public partial string Name { get; set; }` with `[ObservableProperty]`. The older `[ObservableProperty] private string _name` still works but is legacy — use partial properties on new code.
- **`[RelayCommand]` on methods, not `ICommand` properties.** The source generator creates the `ICommand` for you. Name is method name minus "Async" if present.
- **`CancellationToken` parameter on async commands.** The toolkit passes a token that cancels when a new invocation starts (if `IncludeCancelCommand = true`) or when the VM is disposed.
- **`[NotifyCanExecuteChangedFor]`** on properties that affect `CanExecute`. Without it, the command's enabled state doesn't update.
- **Primary constructors for DI** (C# 12+). Clean, no field boilerplate.
- **`sealed` on VMs** unless designed for inheritance. Most aren't.

### What does not go in a ViewModel

- **UI references.** No `Window`, no `Control`, no `Dispatcher` (except through an abstraction). If a VM needs to show a dialog, inject an `IDialogService`.
- **Navigation logic inline.** Route through an `INavigationService` or use the MVVM-style message bus (`WeakReferenceMessenger` from the toolkit).
- **Threading.** Async commands already marshal back to the UI thread via `SynchronizationContext`. Don't call `Dispatcher.UIThread.Post` from a VM.

See `references/viewmodels.md` for validation, messaging, navigation, and VM-to-VM communication.

---

## Views (AXAML)

Compiled bindings only. Declare the ViewModel type:

```xml
<Window xmlns="https://github.com/avaloniaui"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:vm="using:MyApp.ViewModels"
        x:Class="MyApp.Views.MainWindow"
        x:DataType="vm:MainWindowViewModel"
        Title="MyApp"
        Width="600" Height="400">

    <StackPanel Margin="20" Spacing="10">
        <TextBox Text="{Binding Name}" Watermark="Enter your name" />
        <Button Content="Greet"
                Command="{Binding GreetCommand}"
                IsEnabled="{Binding !IsBusy}" />
        <TextBlock Text="{Binding Greeting}" FontSize="18" />
    </StackPanel>
</Window>
```

`x:DataType` + `AvaloniaUseCompiledBindingsByDefault=true` enforces compile-time binding validation. Typos in binding paths fail the build, not the app.

### Rules

- **`x:DataType` on every view.** Without it, bindings silently fall back to reflection.
- **No `ElementName` bindings for cross-control coordination.** Bind through the VM — a button's enabled state is a VM property, not a reference to another control.
- **Styles and resources in `App.axaml`** or in dedicated `.axaml` resource files — not inline per view, unless genuinely scoped.
- **Theme:** `FluentTheme` is the default. `SimpleTheme` exists but is rarely what you want.

### Code-behind

Minimum possible. The generated `InitializeComponent()` is fine. If you find yourself writing real logic in the `.axaml.cs`, it probably belongs in the VM or a behavior.

Acceptable code-behind:
- `InitializeComponent()` call in the constructor.
- Attached behaviors or interactions that genuinely need the control tree (drag-and-drop, canvas manipulation).

Unacceptable:
- Event handlers that update view state — bind instead.
- Navigation logic.
- Data loading.

See `references/axaml-patterns.md` for styles, resources, custom controls, and `DataTemplate`s.

---

## ViewLocator

Convention-based View lookup from ViewModel type. The Avalonia template generates this; customize to use DI.

```csharp
public sealed class ViewLocator : IDataTemplate
{
    public Control? Build(object? data)
    {
        if (data is null) return null;

        var vmType = data.GetType();
        var viewTypeName = vmType.FullName!.Replace("ViewModel", "View", StringComparison.Ordinal);
        var viewType = Type.GetType(viewTypeName);

        if (viewType is null)
            return new TextBlock { Text = $"View not found: {viewTypeName}" };

        return (Control)App.Current.Services.GetRequiredService(viewType);
    }

    public bool Match(object? data) => data is ObservableObject;
}
```

Register in `App.axaml`:

```xml
<Application.DataTemplates>
    <local:ViewLocator />
</Application.DataTemplates>
```

Every View that participates must also be registered in DI (`services.AddTransient<SettingsView>()`).

---

## Threading

Async `[RelayCommand]` methods continue on the UI thread after `await` (captured `SynchronizationContext`). Long work on a background thread is `await Task.Run(...)`:

```csharp
[RelayCommand]
private async Task LoadDataAsync(CancellationToken ct)
{
    IsBusy = true;
    var rows = await Task.Run(() => _repository.LoadAll(ct), ct);
    Items = new ObservableCollection<Row>(rows);  // back on UI thread
    IsBusy = false;
}
```

### Rules

- **Never update `ObservableCollection<T>` from a background thread.** Either post to `Dispatcher.UIThread.Post(...)` or do the update after `await` on the UI thread (preferred).
- **`Dispatcher.UIThread.InvokeAsync` only from services that genuinely can't avoid it.** VMs should not use it directly.
- **Use `IProgress<T>` for progress reporting from background work.** The progress handler runs on the captured context (UI thread) automatically.

---

## Testing ViewModels

ViewModels have no UI reference, so tests don't need a UI thread. Plain xUnit or NUnit.

```csharp
[Fact]
public async Task Greet_UpdatesGreetingFromService()
{
    var service = Substitute.For<IGreetingService>();
    service.GetGreetingAsync("Alice", Arg.Any<CancellationToken>())
           .Returns("Hello, Alice");

    var vm = new MainWindowViewModel(service)
    {
        Name = "Alice"
    };

    await vm.GreetCommand.ExecuteAsync(null);

    vm.Greeting.Should().Be("Hello, Alice");
    vm.IsBusy.Should().BeFalse();
}
```

For commands with `CanExecute` transitions, assert before and after:

```csharp
vm.GreetCommand.CanExecute(null).Should().BeFalse();  // empty Name
vm.Name = "Bob";
vm.GreetCommand.CanExecute(null).Should().BeTrue();
```

### What not to test

- `[ObservableProperty]` firing `PropertyChanged` — that's source-generator code, tested by the toolkit.
- View bindings — they're validated at compile time by `x:DataType`.
- Theme/style rendering — out of scope for unit tests; visual regressions need headless or screenshot testing.

See `references/testing.md` for async command testing, messenger testing, and Avalonia.Headless for UI-level tests.

---

## C# 14 idioms worth using here

- **Partial properties for `[ObservableProperty]`** (Toolkit 8.4+). Cleaner than the field-based legacy syntax.
- **Primary constructors** on VMs for DI.
- **`field` keyword** for properties that do light normalization in the setter without needing an explicit backing field.
- **Collection expressions** (`[]`) for default `ObservableCollection<T>` initialization.
- **`required` members** on message payload records.

---

## What this skill does not cover

- **ReactiveUI.** Different framework, different patterns. If the project uses it, apply ReactiveUI idioms instead — not covered here.
- **Custom rendering / `SkiaSharp` integration.** Niche; use Avalonia docs directly.
- **Mobile / browser targets.** Avalonia supports them but the desktop-specific patterns here (Window, classic lifetime, menu bar) don't translate.
- **Localization.** Straightforward with resource files; not covered as it rarely needs special guidance.
- **Packaging / installers.** For Windows, see the Windows Service skill's deployment reference for `sc.exe`-adjacent patterns; desktop apps typically ship as MSIX, DMG, or AppImage depending on target.
