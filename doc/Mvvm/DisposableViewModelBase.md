# `Mvvm/DisposableViewModelBase.cs`

`DisposableViewModelBase` is the abstract base class for ViewModels that own `IDisposable` resources and need them released deterministically when the ViewModel's lifetime ends.

### Purpose

`ViewModelBase` (defined in `Mvvm/ViewModelBase.cs`) gives every ViewModel in this MVVM stack observable-property support (via Community Toolkit's `ObservableObject`) and child-ViewModel bookkeeping, but says nothing about resource lifetimes. `DisposableViewModelBase` fills that gap: it implements the standard .NET dispose pattern for the case where a ViewModel holds timers, subscriptions, sockets, or similar objects that must be explicitly released, so that owning the resource and cleaning it up come from one place instead of being hand-rolled in every such ViewModel.

It also gives external code a stable type to dispose through: implementing `IDisposableViewModel` (defined in `Mvvm/IDisposableViewModel.cs`, simply `IViewModel, IDisposable`) means any code that holds the ViewModel as an interface reference can call `Dispose()` without knowing the concrete class.

### Role in the Codebase

It is a mid-level building block in the `Mvvm` library, one branch of the ViewModel base-class hierarchy rooted at `ViewModelBase` (its siblings are `MainViewModelBase`, which handles main-window close/shutdown, and `ValidatableViewModelBase<T>`, which adds validation). No class in the repository derives from it or references `Disposables` outside this file, so it is currently infrastructure in place for ViewModels that need disposal; the parallel hierarchy in the docs (`ViewModelBase <|-- DisposableViewModelBase`) treats it as the designated disposal branch.

### Relationships

- Extends `ViewModelBase` (`Mvvm/ViewModelBase.cs`), which extends `ObservableObject` and implements `IViewModel`. All of that is inherited unchanged: `IsInDesignMode`, `OnClosing()`, `OnClosed()`, `RegisterChildViewModel`/`UnregisterChildViewModel`. Nothing here overrides any of it.
- Implements `IDisposableViewModel` (`Mvvm/IDisposableViewModel.cs`), which contributes nothing new itself but makes `Dispose()` reachable through the interface.
- Uses only BCL types (`Collection<T>`, `EventHandler`, `ObjectDisposedException`), no project imports.

### Behavior & Details

- **`Dispose()`** — the public, idempotent entry point. Forwards to `Dispose(true)` and calls `GC.SuppressFinalize(this)`, since the managed path already ran.
- **`Dispose(bool disposing)`** — virtual, so a derived class can add its own cleanup in the standard override pattern. Guarded by `IsDisposed`, so calling it twice (e.g. via `Dispose()` plus the finalizer) has no effect. When `disposing` is true, every object in `Disposables` is disposed, in collection order. Then `IsDisposed` is set, and the `Disposed` event is raised — after the flag is set, so event handlers that inspect state see the ViewModel as already disposed. Note the unmanaged-resource cleanup section of the method is empty; only the `disposing == true` branch does anything today.
- **`Disposables`** — a protected `ICollection<IDisposable>` that derived classes register resources into; it is the only mechanism this class provides for that. The class itself never adds to it.
- **`IsDisposed`** — public, settable only privately. Marked `[Browsable(false)]` so it does not clutter XAML/property-grid design views, consistent with this being a WPF stack.
- **`CheckIfDisposed()`** — protected helper; derived classes call it at the top of methods that touch releasable state to fail fast with `ObjectDisposedException`, named after the concrete runtime type via `GetType().Name`.
- **Finalizer** — calls `Dispose(false)`, i.e. skips the managed `Disposables` loop. The comment notes this is the intended maintainability trade-off, not an oversight.

### Rough edges

- The `Disposed` event is raised while `IsDisposed` is already `true`. A handler that reacts by calling back into the ViewModel (or into anything that calls `CheckIfDisposed()`) will throw — handlers must treat the event as a terminal notification only.
- `Dispose()` is public and unsealed, so a derived class could redeclare or override it and bypass the idempotency and `SuppressFinalize` logic; the contract here only holds for the default implementation.