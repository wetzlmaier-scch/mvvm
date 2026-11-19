# `Mvvm/IDisposableViewModel.cs`

### Purpose

This file defines `IDisposableViewModel`, a marker interface in the `Mvvm` project. It exists so code can tell apart view models that own disposable resources — and therefore must be explicitly `Dispose()`d — from plain view models, without checking the concrete type. The interface body is empty; its meaning comes entirely from the two interfaces it extends.

### Role in the Codebase

The `Mvvm` directory holds the view-model contracts and base classes consumed by the view-layer projects in this solution. `IDisposableViewModel` sits next to the other view-model contracts (`IViewModel`, `IMainViewModel`, `IDialogView`, `IMessageService`) and pairs with the base class that implements it, `DisposableViewModelBase` (`Mvvm/DisposableViewModelBase.cs`). Without it, a view holding a reference to a view model could only dispose it through the concrete type or by trying `IDisposable` on everything; the interface gives external code a stable, semantic type to dispose through.

### Relationships

The interface extends two types, both of which are resolved for this interface:

- `IViewModel` (defined in the same directory, `Mvvm/IViewModel.cs`) — itself just `INotifyPropertyChanged`, the contract every view model in this codebase carries.
- `IDisposable` — the .NET BCL interface, contributing `Dispose()`.

On the implementation side, `DisposableViewModelBase` (`Mvvm/DisposableViewModelBase.cs`) is the only type in the repository that implements `IDisposableViewModel`. It is an abstract class extending `ViewModelBase`, so any concrete view model deriving from it automatically satisfies this interface and is expected to be disposed by whoever holds it — typically a view that created the view model.

### Behavior & Details

There is no behavior to walk through: the interface declares no members of its own, and everything it implies comes from `IViewModel` and `IDisposable`. What a reader can observe is that any `IDisposableViewModel` reference is simultaneously an `INotifyPropertyChanged` and an `IDisposable`, so it can be bound to a view and passed to `Dispose()` in the same expression context. The actual disposal semantics — disposing the `Disposables` collection, setting `IsDisposed`, raising `Disposed`, the `CheckIfDisposed()` guard — live in `DisposableViewModelBase`, not here.