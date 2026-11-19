### Purpose

`MvvmApplication` (in `Mvvm.Wpf/MvvmApplication.cs`) is the WPF `Application` base class that hosts a plug-in-style, module-based WPF application. A concrete app subclasses it — typically with an empty `App : MvvmApplication` — and inherits the full bootstrap: scanning for assemblies, building a MEF composition container, discovering all `IModuleController` implementations, and driving their lifecycle. It also installs the app-wide unhandled-exception handlers, so individual modules never have to.

### Role in the Codebase

The `Mvvm.Wpf` project is a small WPF-targeting library that applications reference; `MvvmApplication` is its application-hosting half (the sibling `MessageService` covers UI messaging). It is the root of the WPF visual tree lifecycle: nothing in a module app runs until `MvvmApplication.OnStartup` composes the container and starts the modules. The sample `WpfApp1/App.xaml.cs` shows the intended usage — a subclass that adds nothing. If this class were removed, a consuming WPF project would have no module discovery, no composition container, and no unified shutdown ordering.

### Relationships

- Inherits from `System.Windows.Application` (WPF framework type). Overrides: `OnStartup`, `OnExit`.
- `Common/Composition/CompositionHelper.cs` — `CreateCompositionContainer` is what actually scans the directory, loads assemblies, and builds the MEF `CompositionContainer` that `MvvmApplication` stores in its `Container` property and exports to itself.
- `Mvvm/IModuleController.cs` — the module contract (`Initialize()`, `Run()`, `Shutdown()`). `MvvmApplication` discovers every `[Export]`-ed implementation from the container and drives the lifecycle.
- `Common/ExitCodes.cs` — supplies `ExitCodes.Error` for the two early-shutdown paths.
- Callers: WPF projects' `App` class (code-behind of `App.xaml`), e.g. `WpfApp1.App`. The primary/secondary/tertiary constructors forward into the one primary constructor taking `moduleAssemblies`, `ignoreAssemblies`, and `searchOption`.

### Behavior & Details

`OnStartup` performs the whole bootstrap, in this order:

1. Subscribes to `DispatcherUnhandledException` and `AppDomain.UnhandledException`.
2. Builds the MEF container via `CompositionHelper.CreateCompositionContainer(moduleAssemblies, ignoreAssemblies, searchOption)`. That helper scans the directory of its own executing assembly for files with the extensions in `AssemblyHelper.Extensions`, plus any `moduleAssemblies` passed in (e.g. from `App.config`), skipping anything in `ignoreAssemblies`, then composes the container against itself.
3. Resolves `Container.GetExportedValues<IModuleController>()`.
4. If `ReflectionTypeLoadException` is thrown (one or more assemblies could not be loaded), it writes each loader exception to `Console` and shuts down with `ExitCodes.Error` — the app exits without further UI.
5. If zero modules were found, it shows the "No module found." message box and shuts down with `ExitCodes.Error`.
6. Calls `Initialize()` on every module, then `Run()` on every module — deliberately in two passes so that all modules are constructed and initialized before any of them starts doing work.

`OnExit` walks `ModuleControllers` in reverse order calling `Shutdown()` on each, then disposes the `Container`, then calls `base.OnExit(e)`. Reverse order means a module that depends on another stops before its dependencies.

Exception handling:

- `AppDispatcherUnhandledException` logs via `Trace.TraceError` and marks the exception handled, so a non-terminating UI-thread exception never crashes the app; it does, however, show a German error dialog (`"Es ist ein Fehler aufgetreten."`) in every culture.
- `AppDomainUnhandledException` is registered unconditionally — even in Debug builds, contrary to the comment at the top of `OnStartup` which says the handlers are only there for non-Debug. The comment only covers the effect that dispatcher-exception handlers are the ones that keep the debugger from breaking on a UI exception; the AppDomain handler is a static subscribe and fires regardless.
- `HandleException` always writes the full exception to the trace; it only shows the dialog when the exception is not terminating.

Constructors: three public overloads plus one primary. The single-parameter one takes only `searchOption` (no module assemblies, no ignore list); the two-parameter one takes an ignore list; the three-parameter primary constructor is the one the others forward to, and it's also the one a subclass can override.

The class carries `[assembly: SupportedOSPlatform("windows")]` at file level, matching the WPF target.

### Rough edges

- The German error message in `HandleException` is hardcoded, so the user-facing dialog is not localizable from this file.
- The comment above the handler subscriptions ("Don't handle the exceptions in Debug mode...") does not match the code: the `AppDomain.UnhandledException` subscription is unconditional and not guarded by a debug check, and even the dispatcher subscription is unconditional in this file.
- When module loading fails via `ReflectionTypeLoadException`, diagnostics go to `Console.WriteLine` — invisible in a default WPF app with no console window — before the app exits.
- `ModuleControllers` is initialized to an empty array and only assigned inside `OnStartup`, so a subclass calling it (or `OnExit` running in the error paths) will iterate zero entries silently; `OnExit` still disposes the container, which is fine, but the reverse-iterate pattern relies on the two-phase bootstrap in `OnStartup` having completed.