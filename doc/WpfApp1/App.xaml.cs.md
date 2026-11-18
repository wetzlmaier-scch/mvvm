# WpfApp1/App.xaml.cs

### Purpose

This file is the code-behalf half of the WPF application's entry point. It defines `WpfApp1.App`, the concrete `Application`-derived class that launches the app. All real work — module discovery, composition, startup, shutdown, and exception handling — is delegated to its base class `MvvmApplication` in the `Mvvm.Wpf` project. The class body here is empty; the file exists so the compiler and XAML tooling can tie the `x:Class="WpfApp1.App"` declaration in `App.xaml` to a C# type that inherits `MvvmApplication` (which itself inherits `System.Windows.Application`).

The file also declares `[assembly: SupportedOSPlatform("windows")]` at the top. Since the project is `net10.0-windows`, this silences the TFM analyzer for Windows-specific APIs used throughout the app; it's an assembly-level attribute, so it effectively applies to the whole project.

### Role in the Codebase

`App` is the executable entry point of the `WpfApp1` project (`OutputType` is `WinExe` in `WpfApp1.csproj`). It is the `partial class` whose other half is generated from `WpfApp1/App.xaml` at build time; the XAML file's root element is `<mvvm:MvvmApplication x:Class="WpfApp1.App">`, so the generated part provides the `Main`-related infrastructure while this source part supplies the class declaration and its base type.

Removing this file (or its class declaration) would break the build, because the XAML-generated partial half has no type to merge with, and the app would have no `Application` subclass to run.

### Relationships

- **`MvvmApplication`** (`Mvvm.Wpf/MvvmApplication.cs`): the base class. `App` overrides nothing and adds no members, so its entire runtime behaviour comes from `MvvmApplication`:
  - On startup it builds a MEF `CompositionContainer` from the assemblies in its directory (minus ignore lists), loads all exported `IModuleController`s, calls `Initialize()` on each, then `Run()` on each.
  - If no modules are found, or a `ReflectionTypeLoadException` occurs while resolving them, it shows an error and shuts down with `ExitCodes.Error`.
  - It hooks `DispatcherUnhandledException` and `AppDomain.UnhandledException`, logging via `Trace.TraceError` and showing a modal error dialog for non-terminating exceptions.
  - On exit it calls `Shutdown()` on the module controllers in reverse order and disposes the container.
- **`WpfApp1.csproj`**: references `Common`, `Mvvm.Wpf`, and `UserManagement`. Because `MvvmApplication`'s default constructor scans the app directory for assemblies to compose, the `UserManagement` project reference is what lets a module shipped in that assembly be discovered and started at runtime. Callers of this class: none directly — the OS/runtime instantiates `App` as the WPF startup object.

### Behavior & Details

- The class is `public partial` and inherits `MvvmApplication` with its default constructor semantics (no module assemblies specified, no ignore list, `SearchOption.TopDirectoryOnly`), meaning module discovery scans only the application's own directory.
- No behavior of its own: no `OnStartup`/`OnExit` overrides, no event handlers, no static members. Anything you'd expect to change at app startup (module set, search scope, ignored assemblies) would go in this class by passing a `MvvmApplication(string[] moduleAssemblies, string[] ignoreAssemblies, SearchOption)`-style argument — currently it isn't.

### Rough edges

- `[assembly: SupportedOSPlatform("windows")]` is duplicated: it appears both here and in `Mvvm.Wpf/MvvmApplication.cs`, which is harmless but redundant.
- The base class's error messages are partly in German (`"Es ist ein Fehler aufgetreten."` / `"Fehler"`), so `App` inherits non-English UI text on exception.