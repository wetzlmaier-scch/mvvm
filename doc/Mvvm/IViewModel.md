### Purpose

**Mvvm/IViewModel.cs** — `Mvvm.IViewModel` is the root marker interface for every view model in this codebase. It exists to give the framework a single type that identifies "this is a view model" and to guarantee that all view models raise `INotifyPropertyChanged` so the UI bindings update automatically.

The interface has no members of its own; its entire substance is its base, `INotifyPropertyChanged` (the .NET BCL type `System.ComponentModel.INotifyPropertyChanged`). Implementing `IViewModel` therefore commits a class to exposing `PropertyChanged`, and that's the whole contract.

### Role in the Codebase

This interface is the root of the view-model type hierarchy in the `Mvvm` project, which is the shared MVVM framework layer that the `Mvvm.Wpf`, `Mvvm.Winforms`, and feature projects (e.g. `UserManagement`) all build on.

The concrete inheritance chain below it:

- `ViewModelBase` (abstract, `Mvvm/ViewModelBase.cs`) implements `IViewModel` and inherits property-change support from the CommunityToolkit `ObservableObject`.
- `MainViewModelBase` and `DisposableViewModelBase` extend `ViewModelBase`; `ValidatableViewModelBase<T>` extends it as well and adds `INotifyDataErrorInfo`.
- The sibling interfaces `IMainViewModel` and `IDisposableViewModel` (`Mvvm/IMainViewModel.cs`, `Mvvm/IDisposableViewModel.cs`) both also extend `IViewModel`, so every main-window and disposable view model is transitively an `IViewModel` too.

If `IViewModel` were removed, the abstract bases could still function through `ObservableObject` alone, but code that resolves or filters "any view model" by type would lose its common denominator. It is the point where the framework says what a view model *is*, distinct from any particular feature's view model.

### Relationships

- **Depends on:** only `System.ComponentModel.INotifyPropertyChanged` — a .NET framework type, no project dependencies.
- **Extenders within this project:** `ViewModelBase`, plus via interface inheritance `IMainViewModel` and `IDisposableViewModel`.
- **Transitive implementers (callers/users):** feature view models such as `UserManagement/ViewModel/UserViewModel.cs` (`UserViewModel : MainViewModelBase, IUserViewModel`) and `ShowUsersViewModel : ViewModelBase, IShowUsersViewModel`. Anyone in the app layer that wants a uniform reference to "a view model" — for DI, lifecycle handling, or child-view-model bookkeeping — would program against `IViewModel`.

### Behavior & Details

There is no behavior to trace: it is an empty interface with one base interface. Two practical consequences for developers writing against it:

- A class implementing `IViewModel` directly (rather than deriving from `ViewModelBase`) must implement `INotifyPropertyChanged` itself, since the framework provides no default.
- Because it is the hierarchy root, any check like `is IViewModel` or a generic constraint on `IViewModel` accepts every view model in the application, including main-window and disposable variants.