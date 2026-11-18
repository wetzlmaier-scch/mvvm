# `Mvvm/IDisposableViewModel.cs`

### Purpose

A marker interface for view models that own resources requiring cleanup. It exists so code can depend on "this view model is disposable" as a distinct contract, without re-spelling both parent interfaces at each call site.

### Role in the Codebase

It is a contract in the `Mvvm` folder, part of a small interface/base-class hierarchy:

- `IViewModel` (defined in `Mvvm/IViewModel.cs`) — an empty interface over `INotifyPropertyChanged`, the marker for "this is a view model".
- `IDisposableViewModel` — `IViewModel` plus `System.IDisposable` (BCL). No new members of its own.
- `DisposableViewModelBase` (defined in `Mvvm/DisposableViewModelBase.cs`) — the abstract class that actually implements the disposal machinery and declares `IDisposableViewModel` explicitly, alongside its inheritance from `ViewModelBase`.

Concretely, `IDisposableViewModel` narrows a type in two ways at once: it is a view model (`IViewModel`), and callers are allowed to call `Dispose()` on it. A view model that holds no resources never needs to implement it. If it were removed, nothing functional breaks immediately — `DisposableViewModelBase` already inherits `IDisposable` through `IDisposableViewModel`, so concrete classes would still be disposable — but the explicit, discoverable contract would be gone, and callers would have to cast to `IDisposable` instead of referencing the type.

### Relationships

- Extends `IViewModel` (same namespace, `Mvvm/IViewModel.cs`) and the BCL `System.IDisposable`.
- Implemented by `DisposableViewModelBase`, which is the only type in the repository that references the interface directly.

Callers who depend on it are code that holds a reference to a possibly-resourceful view model and needs to tear it down — for example view-model holders or navigation code that track which view models must be disposed when they are no longer visible.

### Behavior & Details

There is no behavior to walk through: the interface body is empty, and everything it implies comes from `IViewModel` and `IDisposable`. The only thing a reader can observe is that any `IDisposableViewModel` can be stored in a `Collection<IDisposable>` (like `DisposableViewModelBase.Disposables`, used for child cleanup) or disposed via a common `IDisposable` reference, without knowing the concrete view model type.