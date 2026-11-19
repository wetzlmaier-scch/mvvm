# `Mvvm/IView.cs`

### Purpose

This file defines `IView`, the root of the view-side contract in the `Mvvm` project — the shared, framework-agnostic core of this repository's hand-rolled MVVM setup. Its single member, `object? DataContext { get; set; }`, is the platform-neutral way to attach a view model to a window, so that controllers can wire up a view without knowing whether the host is a WPF `Window` or a WinForms `Form`.

It is a two-line interface. Everything worth knowing about it is context: what extends it, and who assigns through it.

### Role in the Codebase

`IView` is the bottom rung of the view interface hierarchy in `Mvvm`:

- `IView` (`Mvvm/IView.cs`) — declares `DataContext`.
- `IDialogView<TDialogResult>` (`Mvvm/IDialogView.cs`) — adds `DialogResult`, `Closing`/`Closed`, `Show()`, `Close()`.
- `IMainView<TDialogResult, TDimension>` (`Mvvm/IMainView.cs`) — adds the four geometry properties.

Both concrete main windows in the solution — `UserView` (`WpfApp1/View/UserView.xaml.cs`) and `UserForm` (`WinFormsApp1/View/UserForm.cs`) — reach `IView` by implementing `IMainView`. No type in the repository implements `IView` directly.

It is a pure contract with no implementation. Removing it collapses the hierarchy and breaks both view implementations and the two `DefaultModuleController` classes that assign `MainView.DataContext`.

### Relationships

- **Extenders:** `IDialogView<TDialogResult>` (`Mvvm/IDialogView.cs`) is the only direct inheritor; `IMainView` reaches `IView` through it.
- **Implementors:** `UserView` and `UserForm`, via `IMainView`. Both are `[Export]`-ed MEF parts imported by their project's `DefaultModuleController`.
- **Callers:** `WpfApp1/Controller/DefaultModuleController.cs` and `WinFormsApp1/Controller/DefaultModuleController.cs` each do `MainView.DataContext = mainViewModel;` in their constructor, handing the imported `IMainViewModel` to the view through this property.
- **Framework overlap:** In WPF, `DataContext` is inherited from `System.Windows.FrameworkElement` and the interface member is satisfied implicitly. In WinForms, `Form` has no such member, so `UserForm` explicitly overrides `Control.DataContext` (`object`-typed) and uses it to trigger its manual `DataBindings` setup in the setter.

### Behavior & Details

There is no logic in an interface — the interesting behaviour is in how the contract is honoured by the two hosts:

- **WPF (`UserView`):** setting `DataContext` is just `Window.DataContext` assignment, after which WPF's binding engine does the rest. The interface member is satisfied with no extra code.
- **WinForms (`UserForm`):** the `DataContext` setter does double duty. It stores the value, then calls `SetupBindings((IUserViewModel?)value)`, which builds all `DataBindings`, wires `CollectionChanged`/`ErrorsChanged` handlers, and hooks the Save button and form `Load` event. In other words, assigning `DataContext` is what activates the entire WinForms binding layer — the controller's one line `MainView.DataContext = mainViewModel` is the only trigger.

The nullable `object?` type keeps the property compatible with both `Control.DataContext` (`object`) and `Window.DataContext` (`object`) without forcing either side into a cast.

### Rough edges

- In WPF the interface member is satisfied "for free" by `Window.DataContext`, while in WinForms it lands on an override of `Control.DataContext` whose documented purpose is unrelated (arbitrary tag storage). The interface therefore aliases two framework members with different original meanings.
- The setter contract is not uniform: on the WPF side it is a plain assignment, on the WinForms side it performs substantial side-effect work (building bindings). A future view implementor reading only `IView` would not know the setter is expected to be the activation point.