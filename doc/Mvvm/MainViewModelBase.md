# `Mvvm/MainViewModelBase.cs`

`MainViewModelBase` is the abstract base class for the view model bound to the application's main window. It exists so that a concrete main view model gets `IMainViewModel` behaviour — window icon loading, window close/shutdown control, and a closable/cancelable shutdown sequence — without repeating the WPF-specific plumbing.

It is a source file defining a single class: `MainViewModelBase`.

### Purpose

The MVVM hierarchy in this codebase roots feature view models at `ViewModelBase` (which provides `IsInDesignMode` and a child view model registry). The main window's view model needs something more: it owns the window itself, so it must set the window icon, be able to shut the application down with a specific exit code, close the window, and give subclasses a chance to cancel closing. `MainViewModelBase` is where that window-level behaviour lives, factored out of concrete view models.

### Role in the Codebase

It sits in the `Mvvm` project as one of the base-class branches rooted at `ViewModelBase` (its siblings are `DisposableViewModelBase` and `ValidatableViewModelBase`). It is the only class in the repository that implements `IMainViewModel`, so module controllers and the main window's data context resolve `IMainViewModel` and actually get this class's behaviour.

The one concrete subclass is `UserViewModel` (`UserManagement/ViewModel/UserViewModel.cs`), which is exported via MEF as both `IUserViewModel` and `IMainViewModel` and serves as the main window's data context. If this class were removed, `IMainViewModel` would have no implementation, the main window would lose its icon, and `UserViewModel` could no longer be exported as `IMainViewModel`.

### Relationships

- Inherits from `ViewModelBase` (`Mvvm/ViewModelBase.cs`). Inherited: `IsInDesignMode`, the `ObservableObject` binding infrastructure, and the protected `OnClosing()` / `OnClosed()` child-lifecycle hooks. The base class keeps a list of child view models registered via `RegisterChildViewModel` and fans its own `OnClosing()` / `OnClosed()` calls out to them; this class is the only place in the repository that actually triggers those hooks, by translating the window's `Closing` event into them.
- Implements `IMainViewModel` (`Mvvm/IMainViewModel.cs`): `Icon`, `CloseWindow()`, and `Shutdown(int)`.
- Calls `AssemblyHelper.LoadApplicationIcon()` (`Common/Reflection/AssemblyHelper.cs`), which reads the embedded `<entryAssemblySimpleName>.App.ico` manifest resource from the entry assembly and returns a `System.Drawing.Icon`.
- Calls `BitmapSourceHelper.LoadBitmap(Bitmap)` (`Common/Windows/Media/Imaging/BitmapSourceHelper.cs`) to convert that GDI+ icon into a WPF `BitmapSource` for the `Icon` property.

### Behavior & Details

**Constructor** (`protected MainViewModelBase()`):

1. If in design mode, return immediately — no window hook-up, no icon. This matters because designers (and the `devenv` process detection in `ViewModelBase.IsInDesignModeStatic`) construct view models without a running `Application`.
2. If `Application.Current?.MainWindow` is null, return. This keeps the class usable in hosts where there is no main window (for example the console host, where `UserViewModel` still exists but `Application.Current` is absent).
3. Subscribes to `MainWindow.Closing` — this is what activates the otherwise dormant child-view-model lifecycle in `ViewModelBase`.
4. Loads the application icon. If both `LoadApplicationIcon()` and the bitmap conversion succeed, assigns the result to `Icon`; otherwise `Icon` stays null, which the interface declares as acceptable.

Note the side effect: the constructor permanently adds a `Closing` handler to the main window. There is no unsubscribe, so the lifetime of a `MainViewModelBase` instance is effectively tied to the window's lifetime.

**Closing flow:** the private `MainWindow_Closing` handler calls the virtual `OnClosing(CancelEventArgs)`, which:

1. Calls the inherited parameterless `OnClosing()` — the base implementation aggregates over all child view models with `&`, so a single child returning false makes the whole app refuse to close.
2. If that returns false, sets `e.Cancel = true` and the window stays open.
3. Otherwise calls `OnClosed()`, which propagates down to children.

Subclasses override `OnClosing(CancelEventArgs)` (not the parameterless one) if they need to inspect the event or participate in cancellation; the parameterless pair is for the child-tree fan-out.

**`Shutdown(int exitCode)`**: no-op in design mode; otherwise calls `Application.Current?.Shutdown(exitCode)`.

**`CloseWindow()`**: no-op in design mode; otherwise closes `Application.Current?.MainWindow`.

**`Icon`**: `ImageSource?` with a private setter. Nullable by design — set only if the embedded `.App.ico` resource exists and converts successfully.

One caveat: `BitmapSourceHelper.LoadBitmap(Bitmap)` wraps the icon's GDI+ handle in a WPF bitmap source via `Imaging.CreateBitmapSourceFromHBitmap`, and the intermediate `Icon` (and its `Bitmap` inside it) is never explicitly disposed after the `ToBitmap()` call — the HBITMAP handed to WPF is owned by WPF, but the `Icon` object itself goes out of scope without disposal.

### Rough edges

- The design-mode check in the constructor only guards the first line; `Shutdown` and `CloseWindow` guard themselves, but the `Closing` handler and `OnClosing(CancelEventArgs)` do not. In practice the handler is only attached when not in design mode, so this is consistent, just uneven in form.
- The constructor depends on `Application.Current?.MainWindow` already being set at construction time. A `MainViewModelBase` instance created before the main window exists (for example very early in startup, or in a unit test) silently skips the icon and the `Closing` subscription, leaving `Icon` null and the child lifecycle unhooked — with no way to repair that later, since the wiring happens only in the constructor.