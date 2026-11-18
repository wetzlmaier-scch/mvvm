### `WpfApp1/App.xaml.cs`

### Purpose

This file is the partial-code side of the WPF application entry point for the `WpfApp1` application. It does two things: declares the assembly as Windows-only via `[assembly: SupportedOSPlatform("windows")]`, and defines `App`, a partial class that extends `MvvmApplication`. The class body itself is empty — all of its behaviour comes from the base class and from the corresponding `App.xaml` markup, which sets `mvvm:MvvmApplication` as the root element and binds the `x:Class` to `WpfApp1.App`.

### Role in the Codebase

This is the startup entry point of the WPF app. At runtime, the generated `Main` method (from `App.xaml`) instantiates `App`, and `MvvmApplication.OnStartup` takes over: it builds a MEF composition container, discovers exported `IModuleController` instances, and drives them through `Initialize()` / `Run()` (and `Shutdown()` on exit). In this project, the single module discovered is `DefaultModuleController` in `WpfApp1/Controller/DefaultModuleController.cs`, which constructs the main view/view-model pair and shows the main window. Removing this file (and `App.xaml`) would remove the application's entry point entirely.

### Relationships

- **Base class**: `MvvmApplication` in `Mvvm.Wpf/MvvmApplication.cs` (same repository, `Mvvm.Wpf` project). It is a `System.Windows.Application` subclass that implements the MEF-based module bootstrapping described above. `App` overrides nothing and adds no members; it only supplies the application identity.
- **XAML pair**: `WpfApp1/App.xaml` — the other half of the partial class. Its root element is `mvvm:MvvmApplication` with `x:Class="WpfApp1.App"`, so the XAML-generated initializer and this code-behalf file together form the complete `App` class.
- **Consumers**: nothing references `App` directly by name. It is referenced implicitly by the generated entry point and by the XAML. The module it hosts at runtime is `DefaultModuleController`, which is discovered by MEF, not by name.

### Behavior & Details

- `App` itself contributes no logic. It exists so the XAML root (`MvvmApplication`) can be subclassed with an app-local identity, and so the partial type generated from `App.xaml` has a code-side half.
- The `[assembly: SupportedOSPlatform("windows")]` attribute is also applied (identically) in `Mvvm.Wpf/MvvmApplication.cs`; on the app project side it silences the `CA1416` "platform compatibility" analyzer warnings for WPF APIs used by this app's code. Note that this app is an MVVM WPF app, so it is inherently Windows-only regardless of the attribute.
- All startup/shutdown behaviour (exception wiring, MEF container creation, module `Initialize`/`Run`/`Shutdown`, `Container.Dispose`) is inherited from `MvvmApplication` — see that file for details. Notable inherited behaviours: dispatcher and AppDomain unhandled-exception handlers are attached on startup; if no `IModuleController` is discovered, the app shows a "No module found." error box and shuts down; if `ReflectionTypeLoadException` is thrown during export discovery, the app logs the loader exceptions and shuts down.

### Rough edges

- `App` is an empty class with no members; its sole purpose is to exist as the XAML-bound application type. If a reader is looking for startup logic, it lives in `MvvmApplication.OnStartup`, not here.
- The `[assembly: SupportedOSPlatform("windows")]` attribute is duplicated in both `WpfApp1/App.xaml.cs` and `Mvvm.Wpf/MvvmApplication.cs`; the compiler accepts this (same attribute, same args) but it is redundant in one of the two assemblies.