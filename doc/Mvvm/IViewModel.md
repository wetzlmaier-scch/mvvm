I have everything I need. This is a tiny marker interface, so the documentation should stay proportionally short.

### Purpose

**Mvvm/IViewModel.cs** — `Mvvm.IViewModel` is the root marker interface for every view model in this codebase. It declares no members of its own; its entire substance is its base, `INotifyPropertyChanged` (the .NET BCL type `System.ComponentModel.INotifyPropertyChanged`). Implementing it is the codebase's way of saying "this is a view model, and it raises `PropertyChanged` so views can react to its state."

The interface carries no logic, so there is nothing to configure or call. Its value is purely as a common type in the view-model hierarchy.

### Role in the Codebase

It sits at the root of the view-model abstraction in the `Mvvm` project — the shared core that the UI-specific hosts (`Mvvm.Wpf`, `Mvvm.Winforms`, `Mvvm.Console`) and feature code build on. Because it is the top of the hierarchy, any `is IViewModel` check or generic constraint on `IViewModel` accepts every view model in the application, including main-window and disposable variants.

If `IViewModel` were removed, the concrete bases could still work through `ObservableObject` alone, but any code that resolves, casts, or filters "any view model" by this common type would lose its anchor.

### Relationships

- `ViewModelBase` (abstract, `Mvvm/ViewModelBase.cs`) implements `IViewModel` and inherits its property-change support from CommunityToolkit.Mvvm's `ObservableObject`. It is the base most feature view models actually derive from.
- The sibling interfaces `IMainViewModel` (`Mvvm/IMainViewModel.cs`) and `IDisposableViewModel` (`Mvvm/IDisposableViewModel.cs`) both extend `IViewModel`, so every main-window and every disposable view model is transitively an `IViewModel`.
- Downstream implementers include feature view models such as those under `UserManagement/ViewModel/` (for example `UserViewModel` and `ShowUsersViewModel`), which reach `IViewModel` through `MainViewModelBase` / `ViewModelBase`.

### Behavior & Details

There is no behavior to walk through: the body is empty. Two consequences a reader should know:

- A type that implements `IViewModel` directly, rather than deriving from `ViewModelBase`, must implement `INotifyPropertyChanged` itself — the framework supplies no default.
- Any `IViewModel` reference can be treated as an `INotifyPropertyChanged`, so subscribers can always hook `PropertyChanged` on it.

### Rough edges

`ViewModelBase.IsInDesignModeStatic` (in `Mvvm/ViewModelBase.cs`) is where a design-mode detection concern lives, yet it reaches into WPF-specific APIs — casting a `DependencyPropertyDescriptor` metadata default to `bool` and sniffing for the `devenv` process. That is WPF-specific logic sitting in the framework-agnostic `Mvvm` core, and it is a cast that would throw if the metadata default value were not a boolean.