# AXAML Patterns

## Resources and styles

App-wide styles in `App.axaml`:

```xml
<Application xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             x:Class="MyApp.App">
    <Application.Styles>
        <FluentTheme />
        <StyleInclude Source="avares://MyApp/Styles/Buttons.axaml" />
    </Application.Styles>

    <Application.Resources>
        <ResourceDictionary>
            <Color x:Key="AccentColor">#0078D4</Color>
            <SolidColorBrush x:Key="AccentBrush" Color="{DynamicResource AccentColor}" />
        </ResourceDictionary>
    </Application.Resources>
</Application>
```

Per-view resources inside the view:

```xml
<Window.Resources>
    <converters:BoolToVisibilityConverter x:Key="BoolToVis" />
</Window.Resources>
```

**`DynamicResource` for theme-aware values, `StaticResource` for build-time constants.** Theme switches only propagate through `DynamicResource`.

## DataTemplates

Inline for one-off:

```xml
<ItemsControl ItemsSource="{Binding Items}">
    <ItemsControl.ItemTemplate>
        <DataTemplate DataType="models:Person">
            <StackPanel Orientation="Horizontal" Spacing="8">
                <TextBlock Text="{Binding Name}" />
                <TextBlock Text="{Binding Age}" />
            </StackPanel>
        </DataTemplate>
    </ItemsControl.ItemTemplate>
</ItemsControl>
```

App-wide by type, in `App.axaml`:

```xml
<Application.DataTemplates>
    <DataTemplate DataType="models:Person">
        <controls:PersonCard />
    </DataTemplate>
</Application.DataTemplates>
```

The `ViewLocator` handles `ObservableObject`-based lookup; explicit `DataTemplate`s handle model POCOs.

## Compiled bindings

Required for a healthy codebase. Enforced globally via `<AvaloniaUseCompiledBindingsByDefault>true</AvaloniaUseCompiledBindingsByDefault>`.

In AXAML, set `x:DataType` on the root:

```xml
<UserControl x:Class="MyApp.Views.PersonView"
             x:DataType="vm:PersonViewModel">
    <TextBox Text="{Binding Name}" />
</UserControl>
```

Without `x:DataType`, bindings fall back to reflection and lose compile-time checking. The build warns; treat as error.

## Converters

Prefer VM computed properties over XAML converters when possible — easier to test.

```csharp
// In VM
[ObservableProperty] public partial bool IsLoading { get; set; }
public bool IsContentVisible => !IsLoading;
```

```xml
<!-- In view -->
<ContentControl IsVisible="{Binding IsContentVisible}" />
```

For cases where a converter is genuinely the right tool (inverting bool for `IsEnabled`, formatting dates for display), put it in a `Converters/` folder as a stateless singleton.

```csharp
public sealed class InverseBooleanConverter : IValueConverter
{
    public static readonly InverseBooleanConverter Instance = new();

    public object? Convert(object? value, Type targetType, object? parameter, CultureInfo culture)
        => value is bool b ? !b : AvaloniaProperty.UnsetValue;

    public object? ConvertBack(object? value, Type targetType, object? parameter, CultureInfo culture)
        => value is bool b ? !b : AvaloniaProperty.UnsetValue;
}
```

```xml
<Application.Resources>
    <conv:InverseBooleanConverter x:Key="InverseBool" />
</Application.Resources>
```

Avalonia ships with built-in `!` prefix for boolean inversion: `IsEnabled="{Binding !IsBusy}"` — use that before writing a converter.

## Styles

Scoped via selectors:

```xml
<Style Selector="Button.accent">
    <Setter Property="Background" Value="{DynamicResource AccentBrush}" />
    <Setter Property="Foreground" Value="White" />
</Style>

<Style Selector="Button.accent:pointerover /template/ ContentPresenter">
    <Setter Property="Background" Value="{DynamicResource AccentBrush}" />
</Style>
```

Apply via `Classes`:

```xml
<Button Classes="accent" Content="Submit" />
```

Pseudoclasses (`:pointerover`, `:pressed`, `:disabled`) replace WPF's triggers. Don't try to port WPF `Trigger` blocks literally.

## Custom UserControls

Create a `UserControl` when a piece of UI is reused. Expose data via `StyledProperty<T>`:

```csharp
public partial class AvatarView : UserControl
{
    public static readonly StyledProperty<string> NameProperty =
        AvaloniaProperty.Register<AvatarView, string>(nameof(Name), defaultValue: "");

    public string Name
    {
        get => GetValue(NameProperty);
        set => SetValue(NameProperty, value);
    }

    public AvatarView()
    {
        InitializeComponent();
    }
}
```

```xml
<UserControl x:Class="MyApp.Controls.AvatarView"
             x:Name="Root">
    <Border Background="Gray" CornerRadius="20">
        <TextBlock Text="{Binding #Root.Name}" />
    </Border>
</UserControl>
```

`#Root` binds to the named element (the `UserControl` itself). This avoids conflicts with whatever `DataContext` the consumer provides.

## Attached properties and behaviors

For interactions that don't fit the VM (focus handling, drag-drop, validation visuals), use attached properties or `Interaction.Behaviors`:

```xml
xmlns:i="clr-namespace:Avalonia.Xaml.Interactivity;assembly=Avalonia.Xaml.Interactivity"

<TextBox>
    <i:Interaction.Behaviors>
        <behaviors:SelectAllOnFocusBehavior />
    </i:Interaction.Behaviors>
</TextBox>
```

Behaviors go in the `Behaviors/` folder. Write a new one only when VM binding genuinely can't express the intent.

## Keyboard shortcuts

In the window:

```xml
<Window.KeyBindings>
    <KeyBinding Gesture="Ctrl+S" Command="{Binding SaveCommand}" />
    <KeyBinding Gesture="Ctrl+N" Command="{Binding NewCommand}" />
</Window.KeyBindings>
```

For app-wide shortcuts, put them on the main window — Avalonia doesn't have a true app-level accelerator table.

## Localization

`Avalonia.Markup.Declarative` or the standard .NET resource approach. `.resx` files with `IStringLocalizer<T>` injected into VMs.

```xml
<TextBlock Text="{Binding L.GreetingLabel}" />
```

where `L` is `IStringLocalizer<MainWindowViewModel>` exposed as a VM property.

For dev-time English-only apps, skip localization entirely until it's a real requirement. Premature localization is almost always waste.
