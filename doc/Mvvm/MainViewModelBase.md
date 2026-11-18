# Mvvm/MainViewModelBase.cs

### Purpose

`MainViewModelBase` is the abstract base class for the view model that is bound to the application's main window. It exists so that any concrete main view model gets the `IMainViewModel` behaviour — window icon, close-confirmation handling, and app shutdown — for free, without each app reimplementing the plumbing around `Application.Current` and the main window's `Closing` event.

It is a source file defining a single class: `MainViewModelBase`.

### Role in the Codebase

It sits in the `Mvvm` project, the shared MVVM layer that the host projects (`WpfApp1`, `WinFormsApp1`) and feature projects (`UserManagement`) all depend on. It is the middle link in the chain:

- It inherits from `ViewModelBase` (`Mvvm/ViewModelBase.cs`), which supplies `INotifyPropertyChanged` via CommunityToolkit.Mvvm's `ObservableObject`, the `IsInDesignMode` flag, and the child-view-model close cascade (`OnClosing()` / `OnClosed()`).
- It implements `IMainViewModel` (`Mvvm/IMainViewModel.cs`), whose members are `Icon`, `CloseWindow()`, and `Shutdown(int)`.

The concrete main view model in this repository is `UserViewModel` (`UserManagement/ViewModel/UserViewModel.cs`), which extends `MainViewModelBase` and registers itself with MEF via `[Export(typeof(IMainViewModel))]`. `DefaultModuleController` in both host projects imports that export and assigns it as the main view's `DataContext`.

### Relationships

**Inherits `ViewModelBase`** (same project, `Mvvm/ViewModelBase.cs`). What this class inherits from there:

- `IsInDesignMode` — a cached, static design-time check (also true when running under Visual Studio's `devenv` process).
- `OnClosing()` — returns `true`/`false` after asking every registered child view model (`RegisterChildViewModel`) whether it agrees to close.
- `OnClosed()` — walks the registered child view models and tells each it has been closed.

What this class adds is the `IMainViewModel` contract plus the WPF-side wiring:

- `Icon` property: `protected MainViewModelBase()` calls `AssemblyHelper.LoadApplicationIcon()` (`Common/Reflection/AssemblyHelper.cs`), which reads the embedded `<assembly name>.App.ico` manifest resource from the entry assembly as a GDI `Icon`. It then converts it to a WPF `BitmapSource` via `BitmapSourceHelper.LoadBitmap(Bitmap)` (`Common/Windows/Media/Imaging/BitmapSourceHelper.cs`) and assigns the result to the `Icon` property. The `Icon` setter is `private`, so it can only ever be set by this constructor.
- `Shutdown(int exitCode)` and `CloseWindow()`: both delegate to `Application.Current`, so the view model can end the application or close the main window without any host-specific code.
- `MainWindow_Closing` / `OnClosing(CancelEventArgs)`: a private handler subscribes to the main window's `Closing` event in the constructor and routes it into the overridable `OnClosing(CancelEventArgs)` method, which bridges the WPF event to `ViewModelBase`'s child-cascade.

### Behavior & Details

**Constructor** (`protected MainViewModelBase()`):

1. If in design mode, or if `Application.Current?.MainWindow` is null, it returns immediately — no icon loading, no event hookup. That means the icon and the closing hook only ever exist once a main window actually exists at construction time.
2. Subscribes `MainWindow_Closing` to `mainWindow.Closing`.
3. Loads the entry assembly's `.App.ico` and converts it to `Icon`. If there is no embedded icon, `Icon` stays `null`.

Note that `MainWindow_Closing` is subscribed but never unsubscribed, and the constructor holds no reference to the window beyond the event subscription itself.

**`OnClosing(CancelEventArgs)`** (`protected virtual`): the central close-confirmation flow:

1. Calls the inherited parameterless `OnClosing()`, which folds the results of every registered child view model.
2. If any child refused, sets `e.Cancel = true` and stops there — the window stays open and `OnClosed()` is not invoked.
3. Otherwise calls `OnClosed()`, which notifies all registered child view models that the app is closing.

Because `ViewModelBase.OnClosing()` aggregates children with `&` (not short-circuit `&&`), every child gets asked even after one refuses.

**`Shutdown(int exitCode)`**: calls `Application.Current?.Shutdown(exitCode)`. Guarded by `IsInDesignMode` and a null-conditional on `Application.Current`, so it is a no-op in the designer or before the application is created.

**`CloseWindow()`**: closes `Application.Current?.MainWindow` the same guarded way. Unlike `Shutdown`, it does not pass an exit code; the exit code comes from the window's `Close` result.

### Rough edges

- The `Closing` handler is subscribed in the constructor but never unsubscribed, and the icon is only loaded at construction time. Since the view model is created before `DefaultModuleController.Run()` shows the window, a scenario where the main window is replaced after construction would leave `Icon` stale and the old window's `Closing` still wired.
- `Shutdown(int)` and `CloseWindow()` are declared in `IMainViewModel`, but in the host projects `DefaultModuleController` instead calls `MainView.Close()` directly; only the view model's own internal flow drives these members.
- There is no override of `OnClosing(CancelEventArgs)` or `OnClosed()` in the one concrete subclass (`UserViewModel`), so today the class behaves exactly like its base in those paths; the hooks are purely the extension point for future subclasses.