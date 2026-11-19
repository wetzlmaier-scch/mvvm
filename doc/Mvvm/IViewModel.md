# Mvvm/IViewModel.cs

`Mvvm/IViewModel` is the root marker interface for every view model in this codebase. It declares no members of its own; its entire substance is its base, `INotifyPropertyChanged` (from `System.ComponentModel`), so the contract it conveys is simply "this type is a view model that raises property-changed notifications."

### Purpose

In a WPF MVVM setup, views bind to properties and react to `PropertyChanged` events. This interface makes "is a view model" a queryable type rather than a naming convention. It exists so code that wants to handle *any* view model generically — a module controller that wires up the main view model, tests that assert on view-model structure, or DI code keyed by type — has a single root type to target, and so every view model is guaranteed to be change-notifying without restating that base on each sibling interface.

### Role in the Codebase

It sits at the top of the view-model abstraction in the `Mvvm` project — the shared framework-agnostic core that the UI host projects (`Mvvm.Wpf`, `Mvvm.Winforms`, and other feature code such as `UserManagement`) build on. Because it is the common root, the specialized view-model interfaces in this folder all hang off it:

- `IMainViewModel` (`Mvvm/IMainViewModel.cs`) adds window-level members (`Icon`, `CloseWindow()`, `Shutdown(int)`) on top of `IViewModel`.
- `IDisposableViewModel` (`Mvvm/IDisposableViewModel.cs`) combines `IViewModel` with `IDisposable` for view models that own disposable resources.

On the concrete side, `ViewModelBase` (`Mvvm/ViewModelBase.cs`) implements `IViewModel` while inheriting the actual notification plumbing from CommunityToolkit.Mvvm's `ObservableObject`; feature view models such as `ShowUsersViewModel` and `UserViewModel` (under `UserManagement/ViewModel/`) reach the interface through the `ViewModelBase`/`MainViewModelBase` hierarchy. If `IViewModel` were removed, the concrete base classes would still function through `ObservableObject` alone, but any code resolving, casting, or filtering "any view model" by this common type would lose its anchor, and the sibling interfaces would need to re-declare `INotifyPropertyChanged`.

### Relationships

- **Base:** `System.ComponentModel.INotifyPropertyChanged` — a BCL interface, not part of this repository. Inheriting it means every implementer must expose the `PropertyChanged` event.
- **Extenders:** `IMainViewModel` and `IDisposableViewModel`, both in this folder.
- **Implementer:** `ViewModelBase`, in this folder, which is itself the base for `DisposableViewModelBase`, `MainViewModelBase`, and `ValidatableViewModelBase`.
- **Typical callers:** code that takes the interface rather than a concrete base — e.g. `MainViewModelBase` consumers and module controllers that receive the main window's view model and assign it as a WPF `DataContext`.

### Behavior & Details

There is no logic to trace: the interface body is empty, and everything it implies comes from `INotifyPropertyChanged`. Two consequences matter for readers:

- A type that implements `IViewModel` directly, without deriving from `ViewModelBase`, must implement `PropertyChanged` itself — the framework supplies no default implementation.
- Any `IViewModel` reference can be treated as an `INotifyPropertyChanged`, so views and other subscribers can always hook its `PropertyChanged` event.