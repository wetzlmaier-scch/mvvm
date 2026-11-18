# WinFormsApp1/Program.cs

This is a source file defining a single logical unit: the `Program` class, the application's entry point.

### Purpose

`Program` is the bootstrap for the WinForms host of this solution. It exists to do one job: build a MEF (`System.ComponentModel.Composition`) container, pull out every exported `IModuleController`, start them up, and let them drive the UI. All actual application behaviour lives in the module controllers (such as `WinFormsApp1/Controller/DefaultModuleController.cs`); this class only orchestrates their lifecycle and defines what happens when the process is starting up or a crash is occurring.

It also pins the whole assembly to Windows via the `[assembly: SupportedOSPlatform("windows")]` attribute, since the host is WinForms-based.

### Role in the Codebase

This is the executable's entry point (`static int Main`) for the `WinFormsApp1` project. Nothing in the project runs without it: the WinForms window never opens unless `Program.Main` composes the container and calls `Run()` on the module controllers.

The repository contains parallel hosts — the solution also has `WpfApp1` and `ConsoleApp1` directories — which use the same module mechanism through their own entry points. This file is the WinForms half of that pattern: same container building, same two-phase module startup, but `Run()` ends in `Application.Run(...)` of a WinForms view instead of a WPF or console loop.

If this file were removed, the `WinFormsApp1` project would have no entry point and could not build as an executable.

### Relationships

- `Common.Composition.CompositionHelper` (`Common/Composition/CompositionHelper.cs`) — creates the `CompositionContainer`. `Program` calls the parameterless `CreateCompositionContainer()`, which builds an aggregate catalog from all assembly files next to the `Common` assembly (plus the entry assembly) and composes it. The WinForms host therefore relies on the host's `bin` folder containing the assemblies that export views, view models, and module controllers.
- `Mvvm.IModuleController` (`Mvvm/IModuleController.cs`) — the module lifecycle contract: `Initialize()`, `Run()`, `Shutdown()`. The only implementation in this project is `WinFormsApp1.Controller.DefaultModuleController`, exported with `[Export(typeof(IModuleController))]`; it is constructed by MEF with the imported `IMainView<DialogResult, int>` and `IMainViewModel`.
- `Common.ExitCodes` (`Common/ExitCodes.cs`) — supplies the process exit values: `Ok` (`0`) and `Error` (`-1`).
- Standard WinForms/COM+ infrastructure: `ApplicationConfiguration.Initialize()`, `[STAThread]`, `Application.Run` (inside the module controller), `MessageBox`, and the .NET MEF API from `System.ComponentModel.Composition`.

### Behavior & Details

The `Main` flow, in order:

1. `ApplicationConfiguration.Initialize()` applies WinForms app defaults (DPI awareness etc.).
2. `AppDomain.CurrentDomain.UnhandledException` is subscribed to. Despite the adjacent comment claiming exceptions should not be handled in Debug mode, the handler is wired unconditionally — it runs in Debug builds too.
3. The container is created via `CompositionHelper.CreateCompositionContainer()` and kept in the static `_container`.
4. `GetExportedValues<IModuleController>()` is called inside a try/catch. Only `ReflectionTypeLoadException` is caught: in that case each `LoaderExceptions` message is written to the console and the process exits with `ExitCodes.Error` (`-1`). Any other exception (e.g. `CompositionException` from unsatisfied imports) propagates and terminates the process with an unhandled exception.
5. If zero modules were exported, a "No module found." error dialog is shown and the process exits with `ExitCodes.Error`.
6. Two-phase startup: `Initialize()` is called on every module first, then `Run()` on every module. `DefaultModuleController.Initialize()` is empty, and its `Run()` calls `Application.Run((Form)MainView)` — so `Main` blocks in the message loop of the first running module.
7. When the message loop ends, `Main` returns `ExitCodes.Ok` (`0`).

Note that `Shutdown()` is defined by `IModuleController` but is never called by `Program`.

**Unhandled exception handling.** `AppDomainUnhandledException` casts `e.ExceptionObject` to `Exception` and calls `HandleException`, which does two things: logs the full exception via `Trace.TraceError`, and — only if the exception is *not* terminating — shows a modal error box with the German text "Es ist ein Fehler aufgetreten." (An error has occurred). For a terminating exception the process dies after the trace write; the message box is deliberately skipped because it would be useless. This handler only catches exceptions that escaped to the AppDomain level (e.g. from non-UI threads); exceptions thrown on the UI thread are handled by WinForms' own machinery, and a `CompositionException` thrown synchronously in `Main` simply becomes an unhandled startup crash.

### Rough edges

- The comment says exceptions should not be handled in Debug mode, but the `UnhandledException` subscription is not guarded by a Debug/Release check — the comment and the code disagree.
- `Shutdown()` is part of the `IModuleController` contract but `Program` never calls it, so modules are left with no clean shutdown path from the host.
- The unhandled-exception dialog is in German while the "No module found." dialog is in English; UI strings in this host are inconsistently localized.
- The nullable static `_moduleControllers` is only assigned inside the try block; if a `ReflectionTypeLoadException` is caught, the early return is fine, but the code has no defensive handling for `Main` being entered a second time (which the WinForms runtime doesn't do, but the statics give no protection).