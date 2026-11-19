# `Mvvm/ViewModelBase.cs`

### Purpose

`ViewModelBase` is the common base class for the application's MVVM layer. It exists to give every ViewModel, from WPF, three things they would otherwise each reimplement: property-change notification, a design-time detection flag, and a parent/child lifecycle so that closing a container view model propagates through the view models it owns.

It is abstract, so it is always a starting point for derived view models rather than something instantiated directly.

### Role in the Codebase

Within the `Mvvm/` folder, it is the root of the view model hierarchy. The other base classes build on it: `DisposableViewModelBase` adds `IDisposable` semantics and a `Disposables` collection; `MainViewModelBase` adds main-window concerns (`Shutdown`, `CloseWindow`, `Icon`) and hooks into the `OnClosing()`/`OnClosed()` pair defined here; `ValidatableViewModelBase<T>` adds FluentValidation-based error reporting. A concrete example is `ShowUsersViewModel` in `UserManagement/ViewModel/ShowUsersViewModel.cs`, which derives from it directly.

`MainViewModelBase.OnClosing(CancelEventArgs)` is the primary consumer of the lifecycle contract: if `OnClosing()` returns `false`, it cancels the window close, which means any registered child that returns `false` blocks the whole application from closing. Removing this class would break every ViewModel in the project.

### Relationships

- Inherits from `ObservableObject` (CommunityToolkit.Mvvm), which supplies `INotifyPropertyChanged` and the `SetProperty`/`OnPropertyChanged` helpers. This class contributes no property logic of its own; `ObservableObject` provides all of it.
- Implements `IViewModel` (defined in `Mvvm/IViewModel.cs`), which is an empty marker interface extending `INotifyPropertyChanged`.
- Depends on `ExtendedGuard` in `Common/Diagnostics/ExtendedGuard.cs` for the register/unregister preconditions.
- Callers of the protected members are the derived classes above and any view model that manages child view models.

### Behavior & Details

**Design-mode detection**

`IsInDesignMode` is an instance property that simply forwards to the static `IsInDesignModeStatic`. The static value is computed once and cached in `_isInDesignMode`. The first read does two checks:

1. It reads `DesignerProperties.IsInDesignModeProperty` via `DependencyPropertyDescriptor` for `FrameworkElement`.
2. If that reports false, it additionally checks whether the process name starts with `"devenv"` — to catch cases where the designer is running under a Visual Studio developer instance that does not set the WPF design-mode flag.

Because the cache is `bool?`, the check runs exactly once per process; every subsequent property access, on any instance, returns the cached value. Derived classes such as `MainViewModelBase` early-return out of their constructors when `IsInDesignMode` is true, so a lot of the codebase's behaviour implicitly depends on this detection being correct.

**Child view model registration**

`RegisterChildViewModel` and `UnregisterChildViewModel` maintain the `_childViewModels` list and are the only way to add or remove a child. Both guard against the null case (`Guard.IsNotNull`) and against duplicates: registering a child that is already registered, or unregistering one that is not registered, throws `ArgumentException` via `ExtendedGuard.NotContains` / `ExtendedGuard.Contains`.

**Close lifecycle**

`OnClosing` and `OnClosed` are the two halves of the tear-down sequence, both virtual:

- `OnClosing` is asked before close, returning a `bool` — it walks `_childViewModels` with an aggregate using `&` (not `&&`), so every child's `OnClosing()` is consulted even after one returns `false`. The result is `false` if any child refused to close.
- `OnClosed` is called after close and walks the same list recursively, so a grandchild's `OnClosed` runs before its parent's completes, and every level of the tree is visited.

Note that nothing in this class calls these two methods. They are invoked from `MainViewModelBase`, which wires `Application.Current.MainWindow.Closing` to its own `OnClosing(CancelEventArgs)` handler that delegates to the parameterless pair. A plain `ViewModelBase` subclass that is not in the main-window chain gets no lifecycle callbacks at all.

### Rough edges

- `OnClosing`'s use of `&` in the aggregate means a child's `OnClosing()` can be called while the parent's close is already going to be cancelled; if a child's `OnClosing` has side effects, it must be safe to run in that situation.
- `_childViewModels` is an `IList<ViewModelBase>` backed by `List<T>`; iteration during `OnClosed`/`OnClosing` is not protected against a child that unregisters another child mid-traversal.