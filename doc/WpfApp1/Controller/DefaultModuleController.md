# `WpfApp1/Controller/DefaultModuleController.cs`

### Purpose

This file defines `DefaultModuleController`, the WPF host's implementation of the `IModuleController` lifecycle contract. It exists to be discovered by MEF at startup and to compose the application's main window: it binds the main view model to the main view and drives the view's show and close in response to the lifecycle phases the host application invokes.

### Role in the Codebase

The `Controller/` directory in `WpfApp1` contains only this class, making it the single `IModuleController` export in the WPF project. `Mvvm.Wpf/MvvmApplication.cs` (the base class of the WPF host's `App`) resolves all `IModuleController` exports from the MEF container in `OnStartup`, calls `Initialize()` on each, then `Run()` on each; in `OnExit` it calls `Shutdown()` on each, in reverse order. Removing this class would leave the container with no module for the WPF host, and `MvvmApplication.OnStartup` would show a "No module found." error and shut down with `ExitCodes.Error`.

A near-identical sibling, `WinFormsApp1/Controller/DefaultModuleController.cs`, plays the same role in the WinForms host, importing `IMainView<DialogResult, int>` instead of `IMainView<bool?, double>`.

### Relationships

- **`Mvvm.IModuleController`** (`Mvvm/IModuleController.cs`) — the interface implemented here. Declares exactly three methods: `Initialize()`, `Run()`, and `Shutdown()`. All three are overridden in this class as `virtual` with the bodies described below.
- **`Mvvm.IMainView<TDialogResult, TDimension>`** (`Mvvm/IMainView.cs`) — extends `IDialogView<TDialogResult>` in turn, which extends `IView`. Imported here as `IMainView<bool?, double>`; the concrete export is `UserView` (`WpfApp1/View/UserView.xaml.cs`), a WPF `Window` carrying `[Export(typeof(IMainView<bool?, double>))]`.
- **`Mvvm.IMainViewModel`** (`Mvvm/IMainViewModel.cs`) — extends `IViewModel` and declares `Icon`, `CloseWindow()`, and `Shutdown(int exitCode)`. The concrete export is `UserViewModel` (`UserManagement/ViewModel/UserViewModel.cs`), which is exported under both `IUserViewModel` and `IMainViewModel`.
- **Caller** — `MvvmApplication.OnStartup`/`OnExit` (`Mvvm.Wpf/MvvmApplication.cs`) invoke the lifecycle methods through the `IModuleController` interface; nothing references this class by name.

### Behavior & Details

**Construction** — The `[ImportingConstructor]` tells MEF to satisfy two parameters: `IMainView<bool?, double>` and `IMainViewModel`. Beyond storing them in the two `protected` read-only properties, the constructor does one thing that matters for the running app:

```csharp
MainView.DataContext = mainViewModel;
```

This is what makes the `UserView` window actually display the user-management content — the XAML data bindings resolve against `UserViewModel` because of this assignment. If it were removed, the window would open with no bound data.

**`Initialize()`** — Empty. The WPF module has nothing to prepare before it runs.

**`Run()`** — Calls `MainView.Show()`. Since `UserView` is a `Window`, this opens it on the UI thread. Note this is `Show()`, not `ShowDialog()`, so the app keeps running after the call returns; the WPF message loop in the host's `App` keeps the process alive.

**`Shutdown()`** — Calls `MainView.Close()`. Invoked by `MvvmApplication.OnExit` in reverse order over all modules.

**Inheritance of the properties** — `MainView` and `MainViewModel` are `protected`, which is unusual for a class with no known subclasses in this repository. They signal an intent to be subclassed (e.g., a WPF-specific controller that needs the view for extra work), but the access level has no current consumer.

**What is deliberately not done** — The controller never calls the view model's `CloseWindow()` or `Shutdown(int)` members, even though `IMainViewModel` declares them. Closing the window is done directly on the view (`MainView.Close()`), so the view model's own shutdown hook is only reachable from the view model's internals, not from the host's lifecycle path. The same holds in the WinForms sibling.

### Rough edges

- The `bool?` dialog result type parameter on `IMainView<bool?, double>` is vestigial here: `UserView` is a `Window`, never shown as a dialog, so no dialog result is ever produced. The type argument exists only to match the interface's shape.
- `Shutdown()` is defined and called by `MvvmApplication.OnExit`, but `Run()` opens the window with `Show()`, so in practice the application typically exits when the user closes the window itself; the module-level `Shutdown()` path is reached via `OnExit`, which fires from that same window close. There is no separate explicit shutdown trigger from the view model back to the host.