# `Mvvm/IView.cs`

## Purpose

This file defines `IView`, the root of the view-side contract in this repository's hand-rolled MVVM framework. Its single member, `object? DataContext { get; set; }`, is a platform-neutral way to attach a view model to a view: the view model travels as a plain `object?` rather than a typed reference, because `IView` must be implementable by both WPF (`Window`) and WinForms (`Form`), and neither UI framework's base type can reference the other.

## Role in the codebase

`IView` is the base of the view interface hierarchy in the `Mvvm` project (the framework-agnostic core shared by the WPF and WinForms apps):

- `IDialogView<TDialogResult>` (`Mvvm/IDialogView.cs`) extends it and adds `DialogResult`, `Show()`, `Close()`, and `Closing`/`Closed` events.
- `IMainView<TDialogResult, TDimension>` (`Mvvm/IMainView.cs`) extends `IDialogView` and adds `Left`/`Top`/`Width`/`Height`.

Every framework-participating view in the repository derives from `IView` through one of those. It is a shared contract, not an implementation: removing it breaks the hierarchy and therefore every view class and the module controllers that compose them.

## Relationships

- **Implementers (through `IMainView`):** `UserForm` (`WinFormsApp1/View/UserForm.cs`) implements `IMainView<DialogResult, int>`; `UserView` (`WpfApp1/View/UserView.xaml.cs`) implements `IMainView<bool?, double>`. Both are exported via MEF (`[Export(typeof(IMainView<...>))]`) and composed by each app's `DefaultModuleController`.
- **The point of assignment:** each app's `DefaultModuleController` (`WinFormsApp1/Controller/DefaultModuleController.cs`, `WpfApp1/Controller/DefaultModuleController.cs`) does `MainView.DataContext = mainViewModel;` in its constructor. This is the framework's mechanism for handing the view model to the view.

## Behavior & Details

There is no behavior in this file — it is a single-member interface with no default implementations. What matters is how implementations use that one member:

- `DataContext` is untyped and nullable. Nothing validates the type at the contract level; a wrong type surfaces only inside the implementing view.
- In WPF, `DataContext` is inherited from `Window`/`UIElement`, so the property already exists. In WinForms, `UserForm` explicitly `override`s `Control.DataContext` and uses the setter as its setup hook: it stores the value via `base.DataContext = value` and calls `SetupBindings((IUserViewModel?)value)` to wire up its `DataBindings` and event handlers. So "setting `DataContext`" is where a view actually learns it has a view model.

## Rough edges

- `IView`'s `DataContext` collides with members that already exist on `WinForms.Control` and WPF's `UIElement` (with slightly different signatures, `object?` vs `object`). Implementations therefore inherit or override same-named members from different base classes — `UserForm` must explicitly `override` the WinForms version. Whether the interface member is *implemented* or *overridden* is determined by whichever UI framework's base type the implementer chooses, which is invisible to anyone reading `IView` alone.