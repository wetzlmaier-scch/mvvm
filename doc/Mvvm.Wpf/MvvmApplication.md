# Mvvm.Wpf/MvvmApplication.cs

## Purpose

`MvvmApplication` is the base `Application` class for WPF hosts of this codebase's MVVM framework. It exists so a WPF app's `App` class (see `WpfApp1/App.xaml.cs`, which is literally `public partial class App : MvvmApplication` with no body) gets a full application lifecycle for free: composition-container creation, module discovery via MEF, ordered module startup, teardown on exit, and a global exception policy — without any of that code appearing in the consuming app.

It's the WPF counterpart of what `WinFormsApp1/Program.cs` and `ConsoleApp1/Program.cs` do imperatively (both call `CompositionHelper.CreateCompositionContainer(...)` and drive the same `Initialize()` → `Run()` → `Shutdown()` sequence themselves).

## Role in the Codebase

The `Mvvm.Wpf` project is a small library (this file plus `MessageService.cs`) that WPF applications reference. `MvvmApplication` is the piece an app plugs into by making its `App.xaml` root element `mvvm:MvvmApplication` (as `WpfApp1/App.xaml` does) and its code-behind derive from it. The app then supplies behaviour through ME

F exports in its own assemblies rather than code in `App`. Removing it would force every WPF host to re-implement container creation, the two-phase module start, reverse-order shutdown, and the exception handlers.

## Relationships

- **`System.Windows.Application`** (WPF framework type) — the base class. `MvvmApplication` adds nothing visible to WPF itself; it only overrides `OnStartup` and `OnExit` and adds constructors.
- **`Common.Composition.CompositionHelper`** (`Common/Composition/CompositionHelper.cs`) — builds the `CompositionContainer`. It scans the folder containing the *Common* assembly for files with assembly extensions, skips names in `ignoreAssemblies` (case-insensitive), and additionally loads whatever is passed in `moduleAssemblies` via `Assembly.LoadFrom`. A container failure here is caught and only `Console.WriteLine`d, so a WPF app gets no UI feedback from it.
- **`Mvvm.IModuleController`** (`Mvvm/IModuleController.cs`) — the contract with the app's modules: `Initialize()`, `Run()`, `Shutdown()`. Modules opt in with `[Export(typeof(IModuleController))]`, e.g. `WpfApp1/Controller/DefaultModuleController` (which in `Run()` shows the main view, and in `Shutdown()` closes it).
- **`Common.ExitCodes`** (`Common/ExitCodes.cs`) — supplies `ExitCodes.Error` (-1) for the two fatal startup paths.
- **Consumers** — WPF app `App` classes derived from it (the `App` in `WpfApp1`), typically via XAML, since WPF instantiates `Application` from XAML rather than from a constructor call.

## Behavior & Details

**Constructors.** The primary constructor takes `string[] moduleAssemblies`, `string[] ignoreAssemblies`, and an optional `SearchOption` (default `TopDirectoryOnly`), all stored as properties. The two public overloads delegate with an empty `moduleAssemblies`; one takes no arguments, the other only `ignoreAssemblies`. So the default behavior is: scan the app's own directory, exclude nothing, no explicitly named extra assemblies.

Note the parameter order is `moduleAssemblies` *then* `ignoreAssemblies`. The single-array overload is therefore `MvvmApplication(string[] ignoreAssemblies, ...)` — if a caller passes one array, it is treated as *ignore* names, not as assemblies to load. Easy to get backwards.

**Startup (`OnStartup`).** In order:

1. Register the two exception handlers (see below).
2. Create the `CompositionContainer` via `CompositionHelper`.
3. `GetExportedValues<IModuleController>()`. If that throws `ReflectionTypeLoadException`, each loader exception is written to `Console` and the app exits with `ExitCodes.Error` (console output is invisible in a WPF app, so this is effectively a silent early exit).
4. If zero modules were exported, show a "No module found." `MessageBox` and exit with `ExitCodes.Error`.
5. Call `Initialize()` on every module, then — in a second pass — `Run()` on every module. The split means every module is fully initialized before any module's `Run()` executes, which lets a module's `Run()` assume other modules' `Initialize()` already happened.

**Shutdown (`OnExit`).** `ModuleControllers.Reverse().Shutdown()` — modules shut down in the opposite order they started. Then `Container.Dispose()`. If startup failed before any modules ran, this loop is a no-op and the container (possibly null, if `OnStartup` returned before assigning it) is disposed only if non-null.

**Exception handling.** Two handlers cover the usual WPF split:

- `AppDispatcherUnhandledException` catches exceptions on the UI thread, calls the shared handler, then sets `e.Handled = true` — so the app survives any non-terminating UI exception.
- `AppDomain.UnhandledException` catches everything else; it cannot prevent termination.

Both funnel into `HandleException`, which `Trace.TraceError`s the exception and — only when not terminating — shows a modal error box. The UI string here is German ("Es ist ein Fehler aufgetreten." / "Fehler") while the no-modules message above is English, so localisation is inconsistent within the class.

There's a comment claiming the handlers are skipped in Debug mode so the debugger can break on exceptions; the code actually registers them unconditionally, so in Debug a WPF app will still show the error box and swallow UI exceptions rather than breaking.

## Rough edges

- The comment above the handler registration doesn't match the code: handlers are registered in all configurations, so the "don't handle in Debug" intent isn't implemented.
- The `ReflectionTypeLoadException` path reports to `Console`, which a WPF app has no console for — the user sees only a sudden exit.
- The single-`string[]` overload takes `ignoreAssemblies`, the opposite of what the parameter order in the primary constructor suggests; a reader of the signature alone will guess wrong.
- Error message language is mixed (German dialog, English "No module found.").