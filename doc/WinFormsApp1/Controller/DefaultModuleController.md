# WinFormsApp1/Controller/DefaultModuleController.cs

### Purpose

This class is the WinForms application's entry-point controller. It wires together the main form and the main view model via MEF (Managed Extensibility Framework) and drives the application's lifetime: initializing, showing the form, and closing it. It exists so that `Program.Main` can treat the WinForms host as a swappable "module" behind the framework-defined `IModuleController` interface in `Mvvm`.

### Role in the Codebase

The WinForms project is one of several host projects in this repository (a sibling `WpfApp1` contains an analog with the same name). The framework in `Mvvm` (`IModuleController`, `IMainViewModel`, `IMainView<TDialogResult, TDimension>`) is host-agnostic, and each host contributes its own module controller.

`Program.cs` builds a `CompositionContainer`, pulls out every exported `IModuleController` (this class, via its `[Export(typeof(IModuleController))]`), calls `Initialize()` on each, then `Run()` on each. Without this class there would be no exported controller in the WinForms host, and `Main` would exit with the "No module found." error.

### Relationships

- `IModuleController` (defined in `Mvvm/IModuleController.cs`) — the interface this class implements. Its three methods define the application lifecycle: `Initialize()`, `Run()`, `Shutdown()`.
- `IMainView<DialogResult, int>` (defined in `Mvvm/IMainView.cs`) — the WinForms-specific main view interface. In this project, the concrete export is `UserForm` (`WinFormsApp1/View/UserForm.cs`).
- `IMainViewModel` (defined in `Mvvm/IMainViewModel.cs`) — the main view model interface. The concrete export in the currently included projects is `UserViewModel` (`UserManagement/ViewModel/UserViewModel.cs`).
- `Program.cs` — the direct caller: it resolves this class from the composition container and invokes `Initialize()` and `Run()` in that order.

The constructor is marked `[ImportingConstructor]`, so MEF resolves `UserForm` and `UserViewModel` by their respective `[Export]` attributes and injects them.

### Behavior & Details

**Constructor.** Assigns the injected view and view model to the `protected` properties, then sets `MainView.DataContext = mainViewModel`. For `UserForm`, setting `DataContext` is the trigger that builds all of its WinForms `DataBindings` against the view model's sub-view models.

**`Initialize()`** — Declared `virtual` with an empty body. It exists as a hook for derived classes that need to do pre-UI setup (loading configuration, warming caches, etc.). The base implementation does nothing.

**`Run()`** — Calls `Application.Run((Form)MainView)`. This both shows the form and starts the WinForms message loop. It blocks until the form closes, at which point `Main` proceeds to its return. Note that the interface types `IMainView` and `IMainViewModel` both expose `CloseWindow`/`Close`/`Shutdown`, but none of those are invoked here — the lifetime is entirely driven by the user closing the form.

**`Shutdown()`** — Calls `MainView.Close()`. Notably, `Program.cs` never calls `Shutdown()` on any module controller. The form is closed implicitly when the user closes it (which unblocks `Application.Run`), and `Program` exits afterward. The method is therefore only relevant if a future caller of `IModuleController` (e.g. the `Mvvm.Wpf` host) actually drives the shutdown path.

**`virtual` and `protected`.** `Initialize`, `Run`, and `Shutdown` are all `virtual`, and `MainView`/`MainViewModel` are `protected`. This signals the class is designed as a base for module-specific controllers that need access to the main view/view model but want to override the lifecycle behavior. No subclass exists in the current WinForms project; the class is used as-is.

### Rough edges

- `Program.cs` never calls `Shutdown()`, so `Shutdown()` in this class is dead code in the WinForms host. It is only meaningful if some other caller of `IModuleController` invokes the full lifecycle (the `Mvvm.Wpf` host does call `Shutdown()` in reverse order, but that is a different host project).
- `Run()` casts `MainView` to `Form`. This is safe today because the only exported `IMainView<DialogResult, int>` is `UserForm : Form`, but the interface `IMainView<TDialogResult, TDimension>` does not guarantee the underlying type is a `Form`. A non-`Form` implementation would throw an `InvalidCastException` at runtime.
- `IMainViewModel` declares `Shutdown(int exitCode)` and `CloseWindow()`, but this controller never calls them; the view model's own shutdown contract is not exercised from this class.