# `Mvvm/IModuleController.cs`

### Purpose

This file defines `IModuleController`, the lifecycle contract that every application module in this solution must implement. It exists so that the host applications (WPF, WinForms) can discover, start, and stop their feature modules uniformly — each module just implements `Initialize()`, `Run()`, and `Shutdown()` and exports itself through MEF, without the host needing to know anything about the module's internals.

### Role in the Codebase

`IModuleController` is a core abstraction in the `Mvvm` framework project, which holds the shared MVVM interfaces (`IMainViewModel`, `IMessageService`, view-model base types) used by every host. It is the module discovery seam for the whole solution: hosts resolve it from a MEF `CompositionContainer` and drive the application's lifetime through it. If it were removed, the hosts (`MvvmApplication` in `Mvvm.Wpf` and `Program` in `WinFormsApp1`) would have no contract to enumerate modules against, and the plugin-style module architecture would not exist.

### Relationships

- **Implementations** opt in via MEF: `WinFormsApp1/Controller/DefaultModuleController.cs` and `WpfApp1/Controller/DefaultModuleController.cs` both carry `[Export(typeof(IModuleController))]` and implement the three methods. The WPF `DefaultModuleController` wires its `IMainView`/`IMainViewModel` in an `[ImportingConstructor]` and shows the window in `Run()`.
- **Consumers** resolve and drive the lifecycle:
  - `Mvvm.Wpf/MvvmApplication.cs` calls `Container.GetExportedValues<IModuleController>()` in `OnStartup`, then calls `Initialize()` on every controller, `Run()` on every controller, and — in `OnExit` — `Shutdown()` on the controllers in reverse order.
  - `WinFormsApp1/Program.cs` does the same in `Main()` (resolve, `Initialize()`, `Run()`), but never calls `Shutdown()`.

### Behavior & Details

The interface itself is a pure contract — no members, no defaults, no state:

- `Initialize()` — called once per module, on all modules, before any module runs. Modules use it to prepare resources before their UI is shown.
- `Run()` — called once per module, on all modules, after all modules have finished initializing. In the concrete `DefaultModuleController` implementations this is where the main window is shown (blocking for the WinForms host).
- `Shutdown()` — the teardown counterpart; the WPF host guarantees it is called (in reverse startup order during `OnExit`), the WinForms host does not.

The ordering is load-bearing: hosts always call `Initialize()` for every module before `Run()` for any module, so a module can assume other modules are initialized when it runs, and `Shutdown()` runs in reverse order so dependents tear down before their dependencies.

### Rough edges

- `Shutdown()` is only part of a real lifecycle in the WPF host; `WinFormsApp1/Program.cs` never invokes it, so in that host the method is dead code unless a module shuts itself down.
- `Initialize()` and `Run()` are empty in both `DefaultModuleController` implementations; all the actual work happens in the constructor and in `Run()`, so `Initialize()` currently exists only as a lifecycle hook the framework calls.