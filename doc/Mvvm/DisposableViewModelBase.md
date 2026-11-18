# Mvvm/DisposableViewModelBase.cs

### Purpose

`DisposableViewModelBase` is the abstract base class for ViewModels that own `IDisposable` resources (timers, subscriptions, sockets, and similar) and need them released deterministically when the ViewModel goes away. It supplies the classic dispose pattern — a public `Dispose()`, a `Dispose(bool)` that derived classes can override, a `Disposables` collection, a `CheckIfDisposed()` guard, and a `Disposed` event.

### Role in the Codebase

It sits in the `Mvvm` folder alongside the rest of the ViewModel base class family: `ViewModelBase`, `ValidatableViewModelBase<T>`, and `MainViewModelBase`. It is a mixin of the "lifecycle" concern into the inheritance chain, rather than a parallel hierarchy: a concrete ViewModel that needs disposal would derive from this class, gaining both `ViewModelBase` behaviour and disposal support.

Currently no class in the repository derives from it (its only references are its own declaration and finalizer), so it appears to be infrastructure in place for future ViewModels. Its contract is also exposed through `IDisposableViewModel`, so views or other infrastructure can depend on that interface rather than the concrete class.

### Relationships

- Extends `ViewModelBase` (defined in `Mvvm/ViewModelBase.cs`), which itself extends CommunityToolkit.Mvvm's `ObservableObject` and implements `IViewModel`. Everything inherited from `ViewModelBase` — design-mode detection, child ViewModel registration, `OnClosing`/`OnClosed` — is available unchanged here.
- Implements `IDisposableViewModel` (defined in `Mvvm/IDisposableViewModel.cs`), a marker interface that is simply `IViewModel, IDisposable`. That is what makes the `Dispose()` call here the interface implementation.
- Callers would be concrete ViewModels deriving from this class, and any code holding an `IDisposableViewModel` reference and calling `Dispose()` on it.

### Behavior & Details

- Registering resources: add any `IDisposable` to the `Disposables` collection (a protected, read-only `Collection<IDisposable>`). When disposal runs, every entry is disposed in collection order. The base class does not clear or deduplicate the collection; that is left to whoever adds entries.
- Disposal sequence in `Dispose(bool)`:
  1. No-op entirely if `IsDisposed` is already `true` — disposal is idempotent.
  2. If `disposing` is `true` (explicit `Dispose()` call), dispose every item in `Disposables`. If `false` (finalizer path), managed disposables are deliberately left alone.
  3. Set `IsDisposed = true`.
  4. Raise `Disposed`.
- `Dispose()` calls `Dispose(true)` then `GC.SuppressFinalize(this)`, so the finalizer won't run for explicitly disposed instances.
- The finalizer calls `Dispose(false)`, covering only the unmanaged path — there is no actual unmanaged-resource hook in the base, so it exists purely so a future derived override has the full pattern to work with.
- `CheckIfDisposed()` throws `ObjectDisposedException` (named after the concrete runtime type) if called after disposal. Derived classes are expected to call it at the top of methods that touch released resources; nothing in the base enforces that.
- `IsDisposed` is public but has a private setter, so it can be observed from views but only changes via the dispose path. `[Browsable(false)]` hides it from design-time property grids.

### Rough edges

`Disposed` is raised after `IsDisposed` has already been set, so any handler that reacts to the event by calling back into the ViewModel will find `IsDisposed` already `true`, and handlers that run `CheckIfDisposed()` from within `Disposed` will throw.