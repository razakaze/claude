# Testing

## Three tiers

1. **ViewModel unit tests** — plain xUnit/NUnit. No UI thread. 95% of test coverage.
2. **Service tests** — same as above; services are plain classes.
3. **Avalonia.Headless UI tests** — actual rendering, for user-flow validation. Rare.

## ViewModel tests

```csharp
public class MainWindowViewModelTests
{
    [Fact]
    public async Task GreetCommand_PopulatesGreetingFromService()
    {
        var service = Substitute.For<IGreetingService>();
        service.GetGreetingAsync("Alice", Arg.Any<CancellationToken>())
               .Returns("Hello, Alice");

        var vm = new MainWindowViewModel(service) { Name = "Alice" };

        await vm.GreetCommand.ExecuteAsync(null);

        vm.Greeting.Should().Be("Hello, Alice");
    }

    [Fact]
    public void GreetCommand_DisabledWhenNameEmpty()
    {
        var vm = new MainWindowViewModel(Substitute.For<IGreetingService>());

        vm.GreetCommand.CanExecute(null).Should().BeFalse();

        vm.Name = "Alice";

        vm.GreetCommand.CanExecute(null).Should().BeTrue();
    }

    [Fact]
    public async Task GreetCommand_SetsIsBusyDuringExecution()
    {
        var tcs = new TaskCompletionSource<string>();
        var service = Substitute.For<IGreetingService>();
        service.GetGreetingAsync(Arg.Any<string>(), Arg.Any<CancellationToken>())
               .Returns(tcs.Task);

        var vm = new MainWindowViewModel(service) { Name = "Alice" };

        var task = vm.GreetCommand.ExecuteAsync(null);
        vm.IsBusy.Should().BeTrue();

        tcs.SetResult("done");
        await task;

        vm.IsBusy.Should().BeFalse();
    }
}
```

## Testing PropertyChanged

Rarely needed — the source generator is already tested upstream. But for VMs with hand-written `PropertyChanged` logic:

```csharp
[Fact]
public void Name_RaisesPropertyChanged()
{
    var vm = new MainWindowViewModel(Substitute.For<IGreetingService>());
    var changes = new List<string?>();
    vm.PropertyChanged += (_, e) => changes.Add(e.PropertyName);

    vm.Name = "Alice";

    changes.Should().Contain(nameof(MainWindowViewModel.Name));
}
```

## Testing messages

```csharp
[Fact]
public void LoginCommand_PublishesUserLoggedInMessage()
{
    var messenger = new WeakReferenceMessenger();
    var received = new List<UserLoggedInMessage>();
    messenger.Register<UserLoggedInMessage>(this, (_, msg) => received.Add(msg));

    var vm = new LoginViewModel(messenger) { Username = "alice" };
    vm.LoginCommand.Execute(null);

    received.Should().ContainSingle(m => m.Username == "alice");
}
```

Inject `IMessenger` rather than using `WeakReferenceMessenger.Default` — makes tests isolated.

```csharp
services.AddSingleton<IMessenger>(WeakReferenceMessenger.Default);
```

## Async cancellation

```csharp
[Fact]
public async Task LoadCommand_HonorsCancellation()
{
    var service = Substitute.For<IDataService>();
    service.LoadAsync(Arg.Any<CancellationToken>())
           .Returns(async ci =>
           {
               var ct = ci.Arg<CancellationToken>();
               await Task.Delay(1000, ct);
               return Array.Empty<Item>();
           });

    var vm = new DataViewModel(service);

    var loadTask = vm.LoadCommand.ExecuteAsync(null);
    vm.LoadCancelCommand.Execute(null);

    var act = async () => await loadTask;
    await act.Should().ThrowAsync<OperationCanceledException>();
}
```

## Headless UI tests (Avalonia.Headless)

For flows that cross view-VM-service boundaries. Install `Avalonia.Headless.XUnit`:

```xml
<PackageReference Include="Avalonia.Headless.XUnit" Version="11.3.*" />
```

```csharp
public class MainWindowFlowTests
{
    [AvaloniaFact]
    public void TypingInTextBox_EnablesGreetButton()
    {
        var window = new MainWindow { DataContext = new MainWindowViewModel(new StubService()) };
        window.Show();

        var textBox = window.FindControl<TextBox>("NameBox")!;
        var button = window.FindControl<Button>("GreetButton")!;

        button.IsEnabled.Should().BeFalse();

        textBox.Text = "Alice";

        button.IsEnabled.Should().BeTrue();
    }
}
```

Slower than VM tests. Use only for flows that genuinely depend on the UI.

## Design-time data

Not tested — only exists for the designer. Don't assert on it.

## What the tests must cover

- Every `[RelayCommand]`: happy path, `CanExecute` transitions, exception path.
- Every `[ObservableProperty]` with `[NotifyCanExecuteChangedFor]`: assert command state updates.
- Every `ObservableValidator` rule: invalid input produces errors, valid clears them.
- Every message publication and subscription.
- Every service boundary (mock the service, assert VM consumes results correctly).

## What the tests must not do

- Spin up a full `AppBuilder` for every test. Too slow, too brittle.
- Assert on rendered visuals from unit tests — that's what Avalonia.Headless is for.
- Sleep to wait for async work. Use `await` on the returned `Task`.
- Touch `Dispatcher.UIThread` from test code.
