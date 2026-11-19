# `WinFormsApp1/Program.cs`

### Purpose

This file is the entry point of the WinForms host application. Its sole job is to start up the MEF-based module system and hand control to whatever modules the composition container resolves. It has no business logic of its own; it exists so that the `Mvvm` module framework has a process to live in. The same bootstrap pattern exists for the WPF host in `WpfApp1`.

### Role in the Codebase

`Program.cs` is the executable's `Main` — the only code that runs before the module system is up. Everything else in `WinFormsApp1/` (the `Controller/` and `View/` folders) is discovered and started *by* this file through composition, not referenced from it directly. If it were removed, the assembly would not be an application at all.

Its counterpart is `WpfApp1/`, which follows the same create-container → resolve `IModuleController`s → initialize → run flow, but returns an `ExitCodes.Error` via `Application.Shutdown()` instead of a `Main` return value.

### Relationships

- `Common.Composition.CompositionHelper` (`Common/Composition/CompositionHelper.cs`) — builds the `CompositionContainer`. It scans all `*.dll`/`*.exe` files next to the `Common` assembly's location (top directory only, since `Main` calls the parameterless overload), adds each as an `AssemblyCatalog`, plus the entry assembly (`WinFormsApp1` itself). Any load failure is swallowed with a `Console.WriteLine`, so individual assemblies that fail to load do not abort the scan.
- `Mvvm.IModuleController` (`Mvvm/IModuleController.cs`) — the module contract: `Initialize()`, `Run()`, `Shutdown()`. Exports of this interface are the units this host starts.
- `WinFormsApp1.Controller.DefaultModuleController` — the actual in-assembly implementation, marked `[Export(typeof(IModuleController))]`. Its `Run()` calls `Application.Run((Form)MainView)`, so this is also what blocks `Main` until the main form closes.
- `Common.ExitCodes` (`Common/ExitCodes.cs`) — `Ok = 0` and `Error = -1`, used as the process return value.

No other code in the repository calls into `Program`; it is called by the runtime.

### Behavior & Details

`Main` is `[STAThread]` (required for WinForms UI) and returns an `int` used as the process exit code. Startup proceeds in this order:

1. `ApplicationConfiguration.Initialize()` — generated bootstrap for DPI/font settings.
2. An `AppDomain` unhandled-exception handler is attached.
3. The composition container is created via `CompositionHelper.CreateCompositionContainer()`.
4. All exported `IModuleController` values are resolved into the static `_moduleControllers` array.
5. If resolution throws `ReflectionTypeLoadException`, each `LoaderExceptions` entry is printed to the console and `Main` returns `ExitCodes.Error`.
6. If zero controllers were resolved, a "No module found." error dialog is shown and `Main` returns `ExitCodes.Error`.
7. Each controller's `Initialize()` is called — for every controller, before any of them runs.
8. Each controller's `Run()` is called. In the shipped configuration, `DefaultModuleController.Run()` blocks in `Application.Run` until the main form closes, so step 8 is effectively the entire lifetime of the app.
9. When the message loop ends, `Main` returns `ExitCodes.Ok`.

The exception handler (`HandleException`) logs the exception via `Trace.TraceError` and, only if the exception is *not* terminating, shows a German error dialog ("Es ist ein Fehler aufgetreten."). Termination-level exceptions get no dialog; the process simply ends.

### Rough edges

- `IModuleController.Shutdown()` is never called by this host. When the main form closes, `Application.Run` returns and `Main` exits immediately, so `DefaultModuleController.Shutdown()` (which closes the view) is unreachable in practice.
- `Main` returns `ExitCodes.Ok` unconditionally after the `Run()` loop, even if a module's `Run()` returned early or failed silently.
- The exception dialog text is in German while the "No module found." dialog is in English — inconsistent user-facing language within the same host.
- `_container` and `_moduleControllers` are static and never disposed.
- The `ReflectionTypeLoadException` catch will not fire in the usual case where `DefaultModuleController` fails to load: `CompositionHelper` already swallows per-assembly load errors with a `Console.WriteLine`, so the container is simply created without that assembly and the app hits the "No module found." path instead.