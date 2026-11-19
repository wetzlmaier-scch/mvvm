### Purpose

`Mvvm/IModuleController.cs` defines `IModuleController`, the lifecycle contract for an application module. This modular application is composed of one or more "modules" discovered at runtime through MEF (Managed Extensibility Framework) exports. This interface is what the host applications look for: anything exported as `IModuleController` is treated as a module and driven through the same three-phase sequence — `Initialize()`, `Run()`, `Shutdown()`. The interface exists so that hosts can run any number of modules without knowing their concrete types.

### Role in the Codebase

`IModuleController` is a core abstraction in the `Mvvm` framework project, which holds the shared MVVM infrastructure used by all hosts (view-model base classes, view and messaging interfaces). It is the one interface in that project that is not about the view-model layer itself but about the application's overall structure — it is the seam between the host process (WPF or WinForms) and the module that does the actual work.

Removing it would break the discovery mechanism of both hosts: the WPF host's `MvvmApplication` and the WinForms host's `Program.Main` both resolve `GetExportedValues<IModuleController>()` from their composition container and refuse to start if nothing is found (each shows a "No module found." error and exits with `ExitCodes.Error`).

### Relationships

- **Implementers:** `WinFormsApp1/Controller/DefaultModuleController.cs` and `WpfApp1/Controller/DefaultModuleController.cs` both mark their class with `[Export(typeof(IModuleController))]` and implement the interface. Each is the sole module in its host project; its `Run()` starts the main view's message loop (`Application.Run(...)` for WinForms), which is why it effectively blocks until the window closes.
- **Drivers:** `Mvvm.Wpf/MvvmApplication.cs` resolves all exports in `OnStartup`, calls `Initialize()` on every controller, then `Run()` on every controller, and in `OnExit` calls `Shutdown()` on the controllers in reverse order. `WinFormsApp1/Program.cs` does the same `Initialize()`/`Run()` sequence in `Main()`.

A class depends on this interface only if it wants to be picked up as a module by a host; view models and views never reference it.

### Behavior & Details

The interface declares exactly three parameterless, void methods and nothing else:

- `Initialize()` — first phase. Called on every module before any module runs. In the current implementations it is an empty hook that exists so modules can prepare themselves before the UI starts.
- `Run()` — second phase. Called after all modules have been initialized. In `DefaultModuleController` this enters the window's message loop and blocks until the main window closes, so for these hosts `Run()` is where the application "lives".
- `Shutdown()` — teardown phase. The WPF host invokes it on exit (in reverse discovery order, before disposing the container). The WinForms host's `Program.Main` never calls `Shutdown()`, so in that host the method is currently dead code — its `MainView.Close()` only matters if some future path invokes it.

The ordering guarantee the hosts provide — all `Initialize()` calls before any `Run()` call, and `Shutdown()` in reverse order — is the practical contract for anyone writing a new module, even though the interface itself expresses only the three methods.

### Rough edges

`Shutdown()` is part of the contract but is only honored by one of the two hosts; a module written against this interface cannot assume it will be called.