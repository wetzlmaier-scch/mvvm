# Mvvm.Wpf/MvvmApplication.cs

### Purpose

`MvvmApplication` is the base `Application` class for WPF hosts of this codebase's MVVM framework. It exists so a WPF app can launch a MEF-composed, modular application without writing any bootstrap code of its own: a concrete `App` class inherits `MvvmApplication` and the entire lifecycle — assembly scanning, container construction, module discovery, initialization, and teardown — happens in the base class's overridden `OnStartup` and `OnExit`.

### Role in the Codebase

`Mvvm.Wpf` is a small library that WPF applications reference. `MvvmApplication` is the half of it that hosts the app; the sibling `MessageService` handles UI messaging. A WPF project wires it in by making its XAML root element an `MvvmApplication`-derived class. In this solution, `WpfApp1/App.xaml` declares `<mvvm:MvvmApplication x:Class="WpfApp1.App">` and `WpfApp1/App.xaml.cs` is the empty `partial class App : MvvmApplication` — the concrete class overrides nothing, so its complete runtime behaviour comes from this class.

Without this class, WPF hosts would have to replicate the MEF bootstrap sequence themselves (build the container, resolve `IModuleController` exports, drive the initialize/run/shutdown phases). The WinForms host does exactly that by hand in `WinFormsApp1/Program.cs`; `MvvmApplication` is the WPF equivalent packaged as a base class.

### Relationships

- **`System.Windows.Application`** (WPF framework type) — the base class. `MvvmApplication` overrides only `OnStartup` and `OnExit`; everything else about WPF `Application` behaviour is inherited unchanged.
- **`CompositionHelper`** (`Common/Composition/CompositionHelper.cs`) — static helper called in `OnStartup` to build the `CompositionContainer`. It scans the directory of `Common`'s executing assembly for loadable assemblies (minus the ignore list), adds the executing and entry assemblies, loads any explicitly named `moduleAssemblies` with `Assembly.LoadFrom`, and composes the `CompositionContainer` itself into the container as an exported value.
- **`IModuleController`** (`Mvvm/IModuleController.cs`) — the module lifecycle contract with three methods: `Initialize()`, `Run()`, `Shutdown()`. `MvvmApplication` resolves all exported implementations and drives them through those phases.
- **`ExitCodes`** (`Common/ExitCodes.cs`) — supplies `ExitCodes.Error` (`-1`) for the two fatal startup paths.
- **`[assembly: SupportedOSPlatform("windows")]`** — applied to the `Mvvm.Wpf` assembly, marking it as Windows-only. Note `WpfApp1/App.xaml.cs` applies the same attribute again, redundantly.

### Behavior & Details

**Construction.** The primary constructor is `MvvmApplication(string[] moduleAssemblies, string[] ignoreAssemblies, SearchOption searchOption = SearchOption.TopDirectoryOnly)`. Two convenience constructors exist: one taking no arrays and one taking only `ignoreAssemblies`. Both forward to the primary with an empty `moduleAssemblies`. All constructor arguments are captured in read-only `protected` properties, so a derived class can inspect or re-run composition with different settings, but the values are fixed at construction time.

Because XAML-generated code calls the parameterless constructor, WPF hosts that don't override the constructor get an empty `moduleAssemblies` and rely entirely on `CompositionHelper`'s directory scan to find module assemblies.

**Startup (`OnStartup`).** The sequence is:

1. Subscribe to `DispatcherUnhandledException` and `AppDomain.CurrentDomain.UnhandledException`.
2. Build the container via `CompositionHelper.CreateCompositionContainer(ModuleAssemblies, IgnoreAssemblies, searchOption)`.
3. Resolve `GetExportedValues<IModuleController>()`. If that throws `ReflectionTypeLoadException`, each loader exception's message is written to `Console` and the app shuts down with `ExitCodes.Error`. Console output is invisible in a normal WPF app, so in practice the failure is a silent exit with code `-1`.
4. If zero modules were resolved, show a `"No module found."` `MessageBox` and exit with `ExitCodes.Error`.
5. Call `Initialize()` on every module controller, then `Run()` on every module controller. The two-phase loop means every module is fully initialized before any module starts running, so one module's `Run()` can safely depend on another's `Initialize()` having completed.

**Shutdown (`OnExit`).** `Shutdown()` is called on all module controllers in reverse order of startup (via `ModuleControllers.Reverse()`), then the `CompositionContainer` is disposed. This guarantees a module is torn down only after every module that started after it has already shut down.

**Exception handling.** Two private handlers funnel into `HandleException`:

- `AppDispatcherUnhandledException` handles WPF dispatcher exceptions, logs them, sets `e.Handled = true` (so the app does not crash), and shows a German-language `MessageBox` (`"Es ist ein Fehler aufgetreten."`) because `isTerminating` is `false`.
- `AppDomainUnhandledException` handles AppDomain-level exceptions. It also logs, but only shows the `MessageBox` if the exception is *not* terminating — for terminating exceptions the process is ending anyway, so no dialog is shown.

Both handlers log via `Trace.TraceError(e.ToString())`, which in a WPF app goes to the debug output window, not the console.

**Failure modes a caller should know about:**

- If `moduleAssemblies` contains an assembly that is already in the scan set, `CompositionHelper` throws `InvalidOperationException`, which is *not* caught in `OnStartup`. It becomes an unhandled exception during startup.
- `ReflectionTypeLoadException` is the only exception type caught during module resolution. A different exception type (e.g., a malformed MEF export) would propagate up and crash the app before the `MessageBox` path is reached.

### Rough edges

- Fatal startup errors (the `ReflectionTypeLoadException` path) are written to `Console.WriteLine`, which has no visible output window in a WPF application. The user sees a silent exit with code `-1` and no explanation.
- The unhandled-exception `MessageBox` text is in German (`"Es ist ein Fehler aufgetreten."`) while the "No module found" message is in English, so the error surface is inconsistent for non-German-speaking users.
- The single-array convenience constructor `MvvmApplication(string[] ignoreAssemblies, ...)` is easy to confuse with `moduleAssemblies` at the call site, since the parameter is unnamed in the primary constructor's signature.