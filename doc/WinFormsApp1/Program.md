### Purpose

`WinFormsApp1/Program.cs` is the process entry point for the WinForms host of this modular application. Its `Program.Main` wires up the MEF (Managed Extensibility Framework) composition container, discovers and starts the application's modules, and installs a last-resort unhandled-exception handler. Everything interesting the WinForms app does after launch is delegated to exported `IModuleController` implementations; `Program.cs` only orchestrates startup and shutdown of the process itself.

Note that the class is a small piece of infrastructure — the actual UI lives in the modules this file loads.

### Role in the Codebase

- This is the only file in `WinFormsApp1/` that the C# compiler uses to produce an executable; it declares the `Main` entry point that the host runtime invokes.
- It sits alongside `WinFormsApp1/Controller/DefaultModuleController.cs`, which is the module that `Main` will actually run. `Program.cs` does not know that class by name — it only knows the `IModuleController` interface.
- The solution has three parallel host projects (`WinFormsApp1`, `WpfApp1`, and `Mvvm.Wpf`) with near-identical `Main`/`OnStartup` logic; this file is the WinForms member of that family and should be kept in sync with the others when startup semantics change.
- Removing it (or its `Main`) means the WinForms project no longer produces a runnable application. Nothing else in the repo calls `Program.Main`.

### Relationships

- `Common.Composition.CompositionHelper` (`Common/Composition/CompositionHelper.cs`) — used to build the `CompositionContainer`. The no-argument overload scans the directory containing the Common assembly for DLLs, loads each with `Assembly.LoadFrom`, and wraps the set in an `AggregateCatalog`/`AssemblyCatalog` so all types in all loaded assemblies are visible to MEF.
- `Mvvm.IModuleController` (`Mvvm/IModuleController.cs`) — the contract every module implements. `Main` resolves all exports of this interface from the container, then calls `Initialize()` on each, then `Run()` on each. (`Shutdown()` exists on the interface but is never called from `Program.cs` — the WinForms host relies on `Application.Run` returning when the module closes.)
- `Common.ExitCodes` (`Common/ExitCodes.cs`) — supplies `Ok` (`0`) and `Error` (`-1`) as the process return values.
- The module actually resolved at runtime for this host is `WinFormsApp1.Controller.DefaultModuleController`, marked `[Export(typeof(IModuleController))]`. Its `Run()` calls `Application.Run((Form)MainView)`, which is what blocks `Main` until the UI is closed.

### Behavior & Details

1. `Main` is `[STAThread]` and returns an `int`. It first calls `ApplicationConfiguration.Initialize()` (a source-generated method in modern WinForms templates) and then registers `AppDomainUnhandledException` on `AppDomain.CurrentDomain`. The comment claims this is skipped in Debug, but the code registers the handler unconditionally — the handler merely *reports* (a `Trace.TraceError` and, when non-terminating, a German "Es ist ein Fehler aufgetreten." `MessageBox`), it does not swallow the exception, so the debugger can still break on first chance.
2. It creates the container via `CompositionHelper.CreateCompositionContainer()` and stores it in the static `_container` field. `CreateCompositionContainer` silently `Console.WriteLine`s and skips any assembly that fails to load; that is the first reason module resolution can be thin.
3. `_container.GetExportedValues<IModuleController>()` is called inside a `try`/`catch` that catches only `ReflectionTypeLoadException`. In that case each `LoaderExceptions` entry is written to the console and the process returns `ExitCodes.Error`. Any other exception (e.g. `CompositionException` for missing imports) propagates out of `Main` and is handled by the CLR's default policy, not by this file.
4. If zero modules are exported, `Main` shows a "No module found." error dialog and returns `ExitCodes.Error`.
5. Two separate `foreach` loops then run over `_moduleControllers`: first every `Initialize()`, then every `Run()`. This is a deliberate ordering — all modules must be initialized before any is allowed to start, because modules can depend on each other's initialized state.
6. Because `Run()` on `DefaultModuleController` calls `Application.Run(form)`, the second `foreach` blocks on the first module's message loop and never reaches any later `Run()`. With one module that is fine; if a future module is added expecting a second `Run()` to execute, that will not happen as written.
7. When the message loop ends (the main form closes), `Main` falls through and returns `ExitCodes.Ok`. `_container` and `_moduleControllers` are never disposed, and `Shutdown()` is never called on the controllers.

### Rough edges

- The comment above the `AppDomain.CurrentDomain.UnhandledException += ...` line says the handler should not be attached in Debug, but there is no `#if DEBUG` guard — the handler is always attached. It does not suppress first-chance exceptions, so the intended "let the debugger break" behaviour still works, but the comment does not match the code.
- Only `ReflectionTypeLoadException` is caught around module resolution; `CompositionException` (for example, from an unsatisfied `[Import]` in `DefaultModuleController`'s `[ImportingConstructor]`) escapes `Main` as an unhandled exception and produces the default CLR crash output rather than the friendly "No module found." dialog.
- `Shutdown()` is declared on `IModuleController` but never invoked from `Program.cs`, so modules cannot cleanly tear down from the host's side.
- The two `foreach` loops assume each module's `Run()` returns; with `DefaultModuleController` it does not, so only the first module in the array is ever started.
- The user-facing strings are inconsistent: "No module found." is English, the unhandled-exception dialog is German ("Es ist ein Fehler aufgetreten." / "Fehler").