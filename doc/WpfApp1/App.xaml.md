# WpfApp1/App.xaml

### Purpose

This file is the WPF application root definition. It declares the application object as a `MvvmApplication` (from the `Mvvm.Wpf` assembly) instead of the plain `System.Windows.Application`, so that the app's startup is driven by the Mvvm library's module-discovery and composition logic rather than WPF's default startup.

### Role in the Codebase

In `WpfApp1`, this XAML file is the markup half of the `App` class. Its paired code-behind, `WpfApp1/App.xaml.cs`, contains the other half: `public partial class App : MvvmApplication`. The `x:Class="WpfApp1.App"` attribute binds the two together — the XAML instantiates `App`, and the C# partial class supplies the empty body. `App` is the WPF entry point of the application (the `Application`-derived type WPF locates at startup via the generated `Main`), so removing or retargeting this file breaks the app's startup entirely.

There is no application-scope content here: `MvvmApplication.Resources` is an empty element, and notably no `StartupUri` is set. The first window is not opened by this file at all.

### Relationships

- **`MvvmApplication`** (defined in `Mvvm.Wpf/MvvmApplication.cs`) is the root element's base type and the real workhorse. It inherits from `System.Windows.Application` and, in `OnStartup`, builds a `CompositionContainer` that scans for assemblies, resolves every exported `IModuleController`, then calls `Initialize()` on each followed by `Run()`. `OnExit` shuts the controllers down in reverse and disposes the container. `MvvmApplication` is a generic C# primary-constructor type (parameterized by module/ignore assembly lists and a `SearchOption`); instantiating it through XAML like this resolves to its parameterless constructor.
- **`WpfApp1.App`** (`App.xaml.cs`) is the class this XAML instantiates. It adds nothing itself, so the app runs entirely on `MvvmApplication`'s inherited behavior.
- The startup flow then lands in `WpfApp1/Controller/DefaultModuleController.cs`, which is the single `IModuleController` export the composition container discovers in this project. Its `Run()` is what actually shows the first window (`UserView`, via the injected `IMainView`), and `OnExit` triggers its `Shutdown()`.
- `App.xaml` itself is not referenced by any other code — WPF's generated entry point consumes it at startup.

### Behavior & Details

- The `xmlns:mvvm` namespace maps to `clr-namespace:Mvvm.Wpf;assembly=Mvvm.Wpf`, which is how the XAML resolves `MvvmApplication` from the library assembly.
- The file contains no `ShutdownMode`, `StartupUri`, resources, styles, or event handlers. Every runtime behavior a reader might expect at the "App" level — exception hooks, module discovery, the main window lifecycle — lives in the inherited `MvvmApplication.OnStartup`/`OnExit` and in the `IModuleController` implementations.
- One caveat: `MvvmApplication` declares its constructors around array parameters, and XAML here instantiates it without any attached properties or parameters. Which constructor path this resolves to (and therefore what `ModuleAssemblies`/`IgnoreAssemblies` end up being) is not determinable from this file alone; the library's `SearchOption`-defaulted parameterless constructor takes `moduleAssemblies: []`, implying an assembly-directory scan.