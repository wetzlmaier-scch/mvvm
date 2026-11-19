# WpfApp1/App.xaml

The WPF host's application object: `App.xaml` is the XAML definition (root element, no application-level resources), and `App.xaml.cs` is its code-behind (an empty `partial` class plus a `SupportedOSPlatform` attribute). Together the two members compile into a single type, `WpfApp1.App`, which derives from `Mvvm.Wpf.MvvmApplication`.

### Purpose

`App` exists so the WPF host can boot the solution's modular architecture with zero code of its own. All real work — building the MEF container, discovering modules, driving their lifecycle, handling crashes — lives in the base class `MvvmApplication` (in `Mvvm.Wpf/MvvmApplication.cs`). This group is therefore a pure instantiation of that bootstrap: the WPF runtime creates `App` at startup, and the inherited `OnStartup`/`OnExit` do everything.

### Role in the Codebase

`App` is the WPF application entry point. It sits at the root of the `WpfApp1` project alongside `App.config` (holds the SQL connection string), `AssemblyInfo.cs` (theme-info attribute), and the `Controller/` and `View/` folders. Because `App` itself contains no members, the project's actual startup logic lives one level down in `DefaultModuleController` (`WpfApp1/Controller/DefaultModuleController.cs`), which `MvvmApplication` will discover via MEF. Removing this group breaks the WPF host's compilation and means the host no longer has an `Application` subclass, so no container is built and no module ever runs.

### Relationships

- **`Mvvm.Wpf.MvvmApplication`** (`Mvvm.Wpf/MvvmApplication.cs`) — the base class. `MvvmApplication` extends `System.Windows.Application` and, in `OnStartup`, builds a MEF `CompositionContainer` via `CompositionHelper.CreateCompositionContainer` with an **empty** module-assembly list and **empty** ignore list, then resolves `IModuleController` exports, calls `Initialize()` on each, then `Run()` on each; `OnExit` calls `Shutdown()` on each (in reverse) and disposes the container. It also installs `DispatcherUnhandledException` and `AppDomain.UnhandledException` handlers. Because `MvvmApplication`'s `OnStartup` is `protected override` (not `sealed`), the empty `App` class inherits all of this without overriding anything.
- **`WpfApp1.Controller.DefaultModuleController`** — the module that MEF will actually construct and run for this host; it is the single `IModuleController` export in the project.
- **`Common.Composition.CompositionHelper`** — invoked indirectly through `MvvmApplication` to scan every loadable `.dll` next to the entry assembly and build the aggregate catalog.

Nothing else in the solution references `App` directly; the WPF runtime instantiates it through the generated entry point.

### Behavior & Details

- The XAML root element is declared as `mvvm:MvvmApplication` (with `x:Class="WpfApp1.App"`), not `Application`. That is why the `partial class App` in the code-behind can legally declare `: MvvmApplication` — the XAML compiler generates the class against the root element's declared type, and the code-behind adds the `partial` half. The `<mvvm:MvvmApplication.Resources>` element is empty, so the app contributes no application-level resources; views and dialogs must define their own.
- The code-behind contains only the `[assembly: SupportedOSPlatform("windows")]` attribute (a trim/analyzer hint for .NET's platform-conditional API analysis) and the empty class body. There is no constructor, no overridden method, no field.
- At runtime, when `App` is constructed:
  1. `Application`'s base constructor runs.
  2. `MvvmApplication.OnStartup` fires: it wires the exception handlers, builds the MEF container (scanning the output directory), resolves `IModuleController` exports — which for this host yields exactly `DefaultModuleController` — and calls `Initialize()` then `Run()` on it. `Run()` calls `MainView.Show()`, displaying `WpfApp1.View.UserView`.
  3. On two failure paths inside `OnStartup` — a `ReflectionTypeLoadException` during export resolution, or a zero-module resolution — the app shows an error message and shuts down with `ExitCodes.Error`.
  4. `OnExit` shuts down the controllers in reverse and disposes the container.
- Any unhandled exception during the app's lifetime is caught by the inherited handlers: the dispatcher exception is marked handled (the app keeps running), and both paths write to the trace log; a non-terminating exception additionally shows a German-language error dialog ("Es ist ein Fehler aufgetreten.").

### Rough edges

- The base class `MvvmApplication` is generic over which modules to load (it accepts `moduleAssemblies` and `ignoreAssemblies`), but `App` uses the parameterless path, so it loads *every* loadable assembly next to the entry assembly. Any `IModuleController` export in any loaded assembly will be started in this host; there is no way for `App` to narrow that set without overriding `OnStartup`.
- `App` also never uses `App.config`'s `ConnectionString` entry itself — that value is consumed by the data-access layer at runtime, not by this group.