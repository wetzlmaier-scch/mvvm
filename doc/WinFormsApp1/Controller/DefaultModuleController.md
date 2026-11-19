# `WinFormsApp1/Controller/DefaultModuleController.cs`

### Purpose

`DefaultModuleController` is the WinForms host's module controller. It implements the `IModuleController` lifecycle contract (`Initialize` / `Run` / `Shutdown`) from the shared `Mvvm` project and exists to be discovered by MEF: it wires the main window and its view model together, and hosts the WinForms message loop for the application's single module.

### Role in the Codebase

The `Controller/` directory is a single-file folder, and this class is its only occupant. It is the only `IModuleController` implementation in the `WinFormsApp1` project, which makes it the single point where the host process (`Program.cs`) hands control to the MVVM layers. `Program.Main` never references this class by name — it only knows the `IModuleController` contract and resolves all exports from the MEF container. If this class were removed, the WinForms host would fail at startup: `Program.cs` treats "zero module controllers resolved" as an error and exits with `ExitCodes.Error`.

### Relationships

- **`Mvvm.IModuleController`** (`Mvvm/IModuleController.cs`) — the interface implemented here. Declares exactly `Initialize()`, `Run()`, and `Shutdown()` and nothing else. The class opts in via `[Export(typeof(IModuleController))]` so the MEF container can find it without any explicit registration.
- **`Mvvm.IMainView<DialogResult, int>`** (`Mvvm/IMainView.cs`) — the main-window contract (extends `IDialogView<TDialogResult>`, which extends `IView` with its `DataContext` property). The concrete export in this project is `UserForm` (`WinFormsApp1/View/UserForm.cs`), a `Form` carrying `[Export(typeof(IMainView<DialogResult, int>))]`. The cast to `Form` in `Run()` is safe only because of that concrete type.
- **`Mvvm.IMainViewModel`** (`Mvvm/IMainViewModel.cs`) — the main-window view-model contract; it also declares `CloseWindow()` and `Shutdown(int exitCode)`. The concrete export is `UserViewModel` (`UserManagement/ViewModel/UserViewModel.cs`), which carries `[Export(typeof(IMainViewModel))]`.
- **Caller:** `WinFormsApp1/Program.cs`. Its `Main` builds the container with `CompositionHelper`, resolves `GetExportedValues<IModuleController>()`, then calls `Initialize()` on every controller followed by `Run()` on every controller. `Run()` does not return until the message loop ends, after which `Main` returns `ExitCodes.Ok`.

### Behavior & Details

**Construction and wiring.** The `[ImportingConstructor]` tells MEF to satisfy the two parameters from the container: the `UserForm` (as `IMainView<DialogResult, int>`) and the `UserViewModel` (as `IMainViewModel`). Beyond storing them in the two `protected` properties, the constructor performs one real piece of work: `MainView.DataContext = mainViewModel`. In WinForms there is no automatic data binding, so `UserForm` overrides `DataContext` to intercept this assignment and call its own `SetupBindings()` — this single line is what triggers all of the form's binding setup.

**`Initialize()`** — intentionally empty. It exists only to satisfy the contract; any per-module startup logic would go here.

**`Run()`** — starts the WinForms message loop via `Application.Run((Form)MainView)` and blocks until the form is closed. Note this blocks `Program.Main`, so the host never reaches any code after the `Run()` loop in its flow. All three methods are `virtual`, so a subclass could extend the lifecycle without changing the export.

**`Shutdown()`** — calls `MainView.Close()`. This is unreachable in practice: `Program.Main` only ever calls `Initialize()` and `Run()` on controllers, never `Shutdown()`. (The WPF host, by contrast, goes through `MvvmApplication`, which does call `Shutdown()` in `OnExit`.)

### Rough edges

- `Shutdown()` is dead code in this host — nothing in `Program.cs` invokes it, so the module can never be torn down through the lifecycle contract.
- `IMainViewModel` declares `CloseWindow()` and `Shutdown(int exitCode)`, but this controller never calls them. The view model's own shutdown hook — and its ability to signal an exit code back to the host — is therefore never reached from here.