# Mvvm/ViewModelBase.cs

### Purpose

`ViewModelBase` is the abstract root of the application's MVVM view-model hierarchy. It exists so that every view model can share three concerns without each one re-implementing them: change notification (inherited from the CommunityToolkit `ObservableObject`), a parent–child view-model tree that propagates closing/closed events, and a reliable way to know whether the code is running inside the WPF designer.

### Role in the Codebase

This class is the base layer of the Mvvm folder's inheritance ladder:

- `DisposableViewModelBase`, `MainViewModelBase`, and `ValidatableViewModelBase<T>` all derive directly from `ViewModelBase` and add disposal, main-window handling, and FluentValidation support respectively.
- Concrete application view models can derive from it directly (e.g. `ShowUsersViewModel` in `UserManagement/ViewModel/ShowUsersViewModel.cs` does).

It is the piece the rest of the Mvvm infrastructure hangs on: `MainViewModelBase` specifically relies on its `IsInDesignMode` check to skip hooking up the window, and on `OnClosing()`/`OnClosed()` as the closing pipeline it drives from the main window's `Closing` event. Removing this class would break every base class and view model in the project.

### Relationships

- **Inherits `ObservableObject`** (CommunityToolkit.Mvvm, external library): provides property-change notification; concrete view models use its `[ObservableProperty]` support.
- **Implements `IViewModel`** (defined in `Mvvm/IViewModel.cs`): a marker interface that is itself empty except for extending `INotifyPropertyChanged`.
- **`Common.Diagnostics.ExtendedGuard`** (defined in `Common/Diagnostics/ExtendedGuard.cs`): supplies the `Contains`/`NotContains` guard methods; this file also uses the standard `Guard` from CommunityToolkit.Diagnostics.
- **Callers**: the other abstract bases in this folder and all concrete view models that extend them indirectly inherit the child-VM registration and design-mode detection described below.

### Behavior & Details

**Design-mode detection.** `IsInDesignModeStatic` lazily computes once and caches the result in the static `_isInDesignMode`. It reads WPF's standard `DesignerProperties.IsInDesignModeProperty` default value, and additionally treats any process whose name starts with `devenv` (Visual Studio) as design mode, since the designer property can report false while a live preview runs inside `devenv`. Once cached the value never changes, so a process that starts outside the designer will always report `false`. The instance property `IsInDesignMode` is a plain passthrough to the static member.

**Child view-model tree.** Each view model keeps a `List<ViewModelBase>` of registered children:

- `RegisterChildViewModel(viewModel)` adds a child, guarding that the argument is non-null and that it is not already registered (`ArgumentException` in either case).
- `UnregisterChildViewModel(viewModel)` removes a child, guarding that the argument is non-null and that it actually is registered.

**Closing pipeline.** Both methods are `protected virtual` and are intended to be driven by `MainViewModelBase`, which hooks `Application.Current.MainWindow.Closing` and calls them:

- `OnClosing()` folds the children's results with a *bitwise* AND (`&`), so every child's `OnClosing()` is always evaluated — even after one child has already returned `false` — and the result is `false` if any child refuses to close. A view model that needs to veto closing (for example to prompt for unsaved changes) overrides this to return `false`.
- `OnClosed()` walks the registered children in order and calls each child's `OnClosed()`, so cleanup hooks fire recursively through the whole tree.

Note that `MainViewModelBase.OnClosing(CancelEventArgs)` sets `e.Cancel = true` when the pipeline returns `false`, and calls `OnClosed()` only after the window actually closes. There is no automatic cleanup of the `_childViewModels` list — children must be unregistered explicitly, and `OnClosing()`/`OnClosed()` operate on whatever is currently registered.

**Registration invariant.** The guards enforce that a child appears in exactly one parent's list for its lifetime: registering twice or unregistering a child that was never registered throws `ArgumentException`.

### Rough edges

- `ExtendedGuard`'s error messages use `ArgumentException("...'{name}'...")`, but the `name` argument passed by `ViewModelBase` (`nameof(viewModel)`) is never interpolated, so the message literally reads `'{'name'}'`.
- `OnClosed()` has no counterpart that prunes the child list, so after a child view model is disposed the parent still holds a reference to it and will still call into it on the next close.