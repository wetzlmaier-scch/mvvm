### Purpose

`Mvvm/IModuleController.cs` defines `IModuleController`, a three-method lifecycle contract for application modules: `Initialize()`, `Run()`, and `Shutdown()`. Its job is to give every UI host in this solution (the WinForms and WPF apps) one common way to discover and drive all application modules without the host knowing what any specific module does.

### Role in the Codebase

This is a small contract interface in the `Mvvm` project, which holds the shared MVVM abstractions (`IViewModel`, `IMainViewModel`, `IMessageService`, `ViewModelBase`, and so on). `IModuleController` is the module-discovery seam of the whole application: hosts find modules by querying the MEF `CompositionContainer` for exports of this type, then drive them in a fixed order.

The enforced lifecycle is two-phase:

1. `Initialize()` is called on every module first — modules can set up resources, wire dependencies, etc.
2. `Run()` is called on every module only after all modules have initialized, so one module may rely on another's initialized state.
3. `Shutdown()` is called at exit, in the reverse order of the controllers, so dependents tear down before their dependencies.

If this interface were removed, both hosts (`WinFormsApp1/Program.cs` and `Mvvm.Wpf/MvvmApplication.cs`) would have no way to enumerate or start the application's modules.

### Relationships

- **Implementations:** `WinFormsApp1/Controller/DefaultModuleController.cs` and `WpfApp1/Controller/DefaultModuleController.cs` both implement the interface and are registered with MEF via `[Export(typeof(IModuleController))]`. Both make all three methods `virtual`, so a concrete app can derive from `DefaultModuleController` and override the lifecycle steps (e.g. add `Initialize()` work) instead of implementing the interface from scratch.
- **Consumers:** `WinFormsApp1/Program.cs` resolves `[.. _container.GetExportedValues<IModuleController>()]` in `Main()` and calls `Initialize()` on all controllers, then `Run()` on all controllers. `Mvvm.Wpf/MvvmApplication.cs` does the same in `OnStartup`, and additionally calls `Shutdown()` on `ModuleControllers.Reverse()` in `OnExit`.
- **No other dependencies:** the interface has no imports beyond the `Mvvm` namespace it belongs to.

### Behavior & Details

The interface itself contains no logic; the behavior comes from the convention its consumers implement:

- Resolution happens once at startup. Both hosts treat a failed resolution (`ReflectionTypeLoadException`) as fatal — they print the loader exceptions and exit with an error — and they also treat "zero controllers found" as an error rather than starting an empty application.
- Controllers are stored in array order. That order is whatever MEF returns, so there is no explicit way to prioritize one module over another; the reverse-order shutdown simply mirrors that order.
- The WinForms host (`Program.cs`) never calls `Shutdown()`. `Application.Run` blocks inside `Run()` and the process exits when the main form closes, so teardown of modules is left to process termination. Only the WPF host performs the `Shutdown()` pass, in `OnExit`.

### Rough edges

`Shutdown()` is part of the contract, but the WinForms host never invokes it — any cleanup work a WinForms module puts in `Shutdown()` silently never runs.