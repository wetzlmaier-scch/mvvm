# `Mvvm/IDisposableViewModel.cs`

### Purpose

This file defines `IDisposableViewModel`, a marker interface in the `Mvvm` project. It exists to let the codebase single out view models that own disposable resources and therefore must be explicitly disposed. The interface itself adds no new members — its entire meaning is the combination of its two parents: a view model (`IViewModel`) that can be disposed (`IDisposable`).

### Role in the Codebase

The `Mvvm` directory holds the framework-agnostic view-model contracts and base classes of the solution, consumed by the `Mvvm.Wpf` and `Mvvm.Winforms` projects. Within it, `IDisposableViewModel` sits alongside `IViewModel`, `IMainViewModel`, and the abstract base classes. It is the contract that lets the disposal machinery in `DisposableViewModelBase` be described as "this view model is disposable" rather than just "this is a view model".

If it were removed, `DisposableViewModelBase` would lose the type through which views and owners can treat a disposable view model uniformly. Nothing else in the repository currently references it directly, so removing it would compile as long as `DisposableViewModelBase` is adjusted — but the explicit distinction between plain and disposable view models would disappear.

### Relationships

- Inherits from `IViewModel` (defined in `Mvvm/IViewModel.cs`), which itself inherits from `System.ComponentModel.INotifyPropertyChanged`.
- Inherits from `System.IDisposable` (BCL), which contributes the single `Dispose()` member.
- Implemented by `DisposableViewModelBase` (`Mvvm/DisposableViewModelBase.cs`), the only type in the repository that does so. Callers holding an `IDisposableViewModel` reference — typically a view that hosts the view model — are the code that would call `Dispose()` on it.

### Behavior & Details

There is no behavior to walk through: the interface body is empty, and everything it implies comes from `IViewModel` and `IDisposable`. What a reader can observe is that any `IDisposableViewModel` reference is simultaneously:

- a change-notifying view model (settable `INotifyPropertyChanged` contract), and
- disposable, with `Dispose()` the entry point into the pattern implemented in `DisposableViewModelBase` (idempotent `Dispose(bool)`, a `Disposables` collection for owned resources, a `Disposed` event, and a `CheckIfDisposed()` guard).

A view model that holds no unmanaged or disposable resources does not need to implement this interface; a plain `IViewModel`/`ViewModelBase` view model is never expected to be disposed. The interface is the marker that says "this one is".