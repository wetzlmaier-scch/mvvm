# `Mvvm/DisposableViewModelBase.cs`

### Purpose

`DisposableViewModelBase` gives view models a standard way to own disposable resources — event subscriptions, file handles, `CancellationTokenSource`s — and release them in one predictable pass. It layers the standard .NET dispose pattern (public `Dispose()`, protected `Dispose(bool)`, finalizer) on top of the existing `ViewModelBase` and exposes the contract through `IDisposableViewModel`.

### Role in the Codebase

It is an abstract base class in the `Mvvm` directory, the framework-agnostic view-model layer shared by the `Mvvm.Wpf` and `Mvvm.Winforms` projects. Alongside `MainViewModelBase` and `ValidatableViewModelBase`, it is one of the specialised bases built on `ViewModelBase`, for view models whose lifetime must be explicitly ended.

No concrete view model currently derives from it, and nothing in the repository populates its `Disposables` collection or calls `CheckIfDisposed()`. It exists as ready infrastructure: when a view model needs to clean up resources, it inherits this class and gets the machinery for free. Removing it would break nothing today, but it would remove the intended home for disposable view models.

### Relationships

- Extends `ViewModelBase` (`Mvvm/ViewModelBase.cs`). Inherited from there: `ObservableObject` (property change notification), design-mode detection, and the `OnClosed()`/`OnClosing()`/`RegisterChildViewModel()`/`UnregisterChildViewModel()` child-view-model machinery. None of that is overridden here.
- Implements `IDisposableViewModel` (`Mvvm/IDisposableViewModel.cs`), which is itself just `IViewModel, IDisposable` — an empty marker. The only substance is the `IDisposable` surface this class provides, and the marker lets views or controllers hold a reference and call `Dispose()` without depending on the concrete type.
- Expected callers are views or view controllers that create a view model, hold it as an `IDisposableViewModel`, and dispose it when the UI goes away.

### Behavior & Details

- `Disposables` is a protected, read-only `Collection<IDisposable>` initialised inline. Derived view models add their resources to it; that is the intended registration point.
- `Dispose()` calls `Dispose(true)` and then `GC.SuppressFinalize(this)`, the standard public entry point.
- `Dispose(bool disposing)` is guarded by `IsDisposed`, so any number of calls (public `Dispose()`, the finalizer, a base-class call) results in exactly one cleanup pass:
  1. When `disposing` is `true`, every item in `Disposables` is disposed, in the order it was added.
  2. `IsDisposed` is set to `true` unconditionally (the comment marks this spot for unmanaged cleanup, though no unmanaged resources are actually held).
  3. The `Disposed` event fires, once.
- `IsDisposed` has a private setter and `[Browsable(false)]`, so it is settable only inside this class and hidden from property editors in designers.
- `CheckIfDisposed()` throws `ObjectDisposedException` (named for the concrete subclass, via `GetType().Name`) when called after disposal. Derived classes can gate their members with it — but nothing in the repository calls it yet.
- The finalizer calls `Dispose(false)`, so if the GC reclaims a view model before `Dispose()` was called, no managed items in `Disposables` get disposed (they are only touched when `disposing` is `true`) and `IsDisposed` is still set. The finalizer is a safety net, not a cleanup path.

### Rough edges

- `Dispose(bool)` iterates `Disposables` without clearing or removing it; if a subclass overrode the method and added items after calling `base`, the `IsDisposed` guard makes a second pass impossible, and any `Dispose()` implementation in `Disposables` that throws will abort disposal of the remaining items with no try/finally around them.
- The `Disposed` event is raised inside `Dispose(bool)` rather than only from the public `Dispose()`, so a subclass that overrides `Dispose(bool)` and calls `base` while still adding to `Disposables` in its own override risks disposing items twice — the guard prevents a second full pass, but the single pass already ran for items present at base-call time.