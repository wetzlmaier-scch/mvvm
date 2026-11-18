# WinFormsApp1/Controller/DefaultModuleController.cs

### Purpose

`DefaultModuleController` is the WinForms host's module controller. It implements the `IModuleController` lifecycle contract from the shared `Mvvm` project and exists to be discovered by MEF: it wires the application's main form (`UserForm`) to its view model (`UserViewModel`) and then starts the WinForms message loop. Without it, `Program.Main` would find no `IModuleController` exports from this assembly and exit with "No module found."

A sibling class with the same name in `WpfApp1/Controller/DefaultModuleController.cs` plays the identical role for the WPF host, substituting `IMainView<bool?, double>` for the WinForms-specific `IMainView<DialogResult, int>`.

### Role in the Codebase

The `Controller` directory is a single-file folder, and this class is its only occupant. It is the only `IModuleController` implementation in the `WinFormsApp1` project, which makes it the single point through which `Program.Main` drives the app's lifecycle:

1. `Program.Main` builds a MEF `CompositionContainer` via `CompositionHelper.CreateCompositionContainer()`.
2. `GetExportedValues<IModuleController>()` finds this class through its `[Export(typeof(IModuleController))]` attribute and constructs it, using the `[ImportingConstructor]` to satisfy its two parameters from other MEF exports.
3. `Program.Main` calls `Initialize()` on every controller, then `Run()` on every controller. `Run()` here blocks for the whole lifetime of the app.
4. `Program.Main` then returns `ExitCodes.Ok`.

The protected `MainViewModel` and `MainView` properties exist so a future derived controller could reach the wired-up view and view model; the three virtual methods (`Initialize`, `Run`, `Shutdown`) are the extension points for customizing the lifecycle.

### Relationships

- **`Mvvm.IModuleController`** (`Mvvm/IModuleController.cs`) — the interface implemented here. Declares exactly `Initialize()`, `Run()`, `Shutdown()` and nothing else.
- **`Mvvm.IMainView<TDialogResult, TDimension>`** (`Mvvm/IMainView.cs`) — a WinForms-era abstraction over the main window. This class imports it instantiated as `IMainView<DialogResult, int>` (WinForms `DialogResult` and `int` for position/size). It extends `IDialogView<TDialogResult>`, which is where the `DataContext` property comes from, alongside `DialogResult`, `Show()`, and `Close()`. The concrete export the MEF container injects is `UserForm` (`WinFormsApp1/View/UserForm.cs`), which carries `[Export(typeof(IMainView<DialogResult, int>))]` and derives from `Form`.
- **`Mvvm.IMainViewModel`** (`Mvvm/IMainViewModel.cs`) — the main window's view model contract; it also declares `CloseWindow()` and `Shutdown(int exitCode)`. The concrete export is `UserViewModel` (`UserManagement/ViewModel/UserViewModel.cs`), tagged `[Export(typeof(IMainViewModel))]` (and `[Export(typeof(IUserViewModel))]`).
- **`UserForm.DataContext`** — assigning `MainViewModel` to `MainView.DataContext` is what actually activates the UI: `UserForm` overrides the `DataContext` setter and, when it receives a non-null `IUserViewModel`, calls `SetupBindings(...)` to hook up its text boxes, list box, error provider, and button to the view model's properties and commands.

### Behavior & Details

**Constructor** — MEF calls the `[ImportingConstructor]` when composing. It stores both dependencies in the read-only `MainViewModel`/`MainView` properties and sets `MainView.DataContext = mainViewModel`. The `DataContext` assignment is the only observable side effect at construction: it is what makes the form's controls bind to `UserViewModel`. Because `UserForm` is the sole `IMainView<DialogResult, int>` export in this project, the `DataContext` assignment goes through the override that calls `SetupBindings`.

**`Initialize()`** — empty. The contract's "before-run" phase has nothing to do in this app, but the method exists (as `virtual`) so a derived controller can extend it without changing `Program.Main`.

**`Run()`** — `Application.Run((Form)MainView)`. This shows the main form and starts the WinForms message loop; the call blocks until the form closes, which is what ends the application. Note the explicit cast: `IMainView<DialogResult, int>` does not constrain the implementing type to be a `Form`, so the interface alone does not make this safe. Today the only export is `UserForm : Form`, so the cast holds — adding a non-`Form` implementation of `IMainView<DialogResult, int>` would fail here with an `InvalidCastException`.

**`Shutdown()`** — calls `MainView.Close()`. `Program.Main` never invokes `Shutdown()` on any controller (it calls only `Initialize()` and `Run()`), so in this host the method is currently dead code. The form closes naturally when the user closes it, which unblocks `Run()`. The method would only matter for a hypothetical caller that drives the full `IModuleController` lifecycle (as `MvvmApplication` in `Mvvm.Wpf` does with `Reverse()` shutdown ordering).

**View model lifecycle gap** — `IMainViewModel` declares `CloseWindow()` and `Shutdown(int exitCode)`, but this controller never calls them. The view model's own shutdown hook is not reached from here; the `int` return value from `Main()` in `Program` is a fixed `ExitCodes.Ok` regardless of how the form closed.

### Rough edges

- The `(Form)MainView` cast in `Run()` is guaranteed only by there being exactly one `IMainView<DialogResult, int>` export in this project. A second export of a non-`Form` type would break the cast.
- `Shutdown()` is defined but never invoked by `Program.Main`, so the only controller-level shutdown path in the WinForms host is the implicit form close.
- `IMainViewModel`'s `CloseWindow()` and `Shutdown(int)` are never called from this controller, so the view model cannot signal an exit code back to the host through the interface it advertises.