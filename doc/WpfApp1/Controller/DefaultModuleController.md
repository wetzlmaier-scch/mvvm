# WpfApp1/Controller/DefaultModuleController.cs

### Purpose

`DefaultModuleController` is the WPF application's bootstrapping controller. It implements `IModuleController` (defined in `Mvvm/IModuleController.cs`) with the minimal lifecycle the app needs: wire the main view to its view model, show the window, and close it on exit. It exists so the WPF host application has a concrete `IModuleController` export that `MvvmApplication` can discover and drive at startup.

### Role in the Codebase

The WPF app's host is `MvvmApplication` (`Mvvm.Wpf/MvvmApplication.cs`), which derives from `System.Windows.Application`. During `OnStartup` it builds a MEF `CompositionContainer`, loads all exported `IModuleController` values, calls `Initialize()` on each, then `Run()` on each. During `OnExit` it calls `Shutdown()` in reverse order. This class is one of the controllers in that list — in this project, the only `IModuleController` export in the `WpfApp1` assembly, so it is effectively the single boot path of the WPF app.

Its counterpart in the WinForms project, `WinFormsApp1/Controller/DefaultModuleController.cs`, plays the same role with WinForms types (`IMainView<DialogResult, int>`).

If this class were removed, the container would find no `IModuleController` exports from `WpfApp1` and `MvvmApplication` would show "No module found." and shut down.

### Relationships

- **MEF wiring.** The class carries `[Export(typeof(IModuleController))]`, and its constructor is marked `[ImportingConstructor]`. The MEF container (created in `MvvmApplication.OnStartup`) satisfies the two constructor parameters:
  - `IMainView<bool?, double>` — exported by `UserView` (`WpfApp1/View/UserView.xaml.cs`), a `Window` that also exports `IMainView<bool?, double>`.
  - `IMainViewModel` — exported by `UserViewModel` (`UserManagement/ViewModel/UserViewModel.cs`), which carries `[Export(typeof(IMainViewModel))]` in addition to its `IUserViewModel` export.
- **Interfaces.** It implements `IModuleController` from `Mvvm/IModuleController.cs` (`Initialize`, `Run`, `Shutdown`). `IMainView<TDialogResult, TDimension>` comes from `Mvvm/IMainView.cs` and inherits from `IDialogView<TDialogResult>`; here the dialog result is `bool?` and dimensions are `double`, matching WPF's unitless, double-based geometry. `IMainViewModel` comes from `Mvvm/IMainViewModel.cs`.

### Behavior & Details

- **Constructor.** Stores the injected `mainView` and `mainViewModel` in the protected `MainView` and `MainViewModel` properties, and sets `MainView.DataContext = mainViewModel`, binding the window to its view model. Everything else in the class is lifecycle plumbing.
- **`Initialize()`.** Empty. Kept as an override point — a subclass can do module setup (e.g. loading data, preparing services) before the view is shown.
- **`Run()`.** Calls `MainView.Show()`, displaying the main window. This is what actually makes the app visible.
- **`Shutdown()`.** Calls `MainView.Close()` to tear down the window.
- All three methods are `virtual`, so a derived controller can extend the lifecycle without changing how `MvvmApplication` drives it.

Because `MainView` and `MainViewModel` are `protected` rather than `public`, they are exposed for derived controllers to use, not for external callers. Callers of this class are the MEF container (construction) and `MvvmApplication` (lifecycle calls) only.

### Rough edges

- `Run()` uses `Show()`, not `ShowDialog()` or `Show()` followed by a result read, even though the view's dialog-result type is `bool?`. The `bool?` result is never observed anywhere in this controller or `MvvmApplication`; the dialog-result generics appear to be carried over from the `IDialogView` contract rather than used by the boot path.
- The WPF app relies on exactly one `IMainViewModel` export (`UserViewModel`). If more than one assembly exporting `IMainViewModel` were composed in, the constructor's `IMainViewModel` import would fail at composition time — there is no named or constrained import here to disambiguate.