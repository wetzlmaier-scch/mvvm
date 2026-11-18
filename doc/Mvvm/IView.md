# `Mvvm/IView.cs`

## Purpose

This file defines `IView`, the root of the view-side contract in this repository's hand-rolled MVVM framework. Its single member, `object? DataContext { get; set; }`, gives the framework a platform-neutral way to attach a view model to a view: the view model is carried as a plain `object?` rather than a typed reference, because `IView` must be implementable by both WPF (`Window`) and WinForms (`Form`), and neither UI framework's base type can reference the other.

## Role in the codebase

`IView` is the base marker for the view interface hierarchy in the `Mvvm` project:

- `IDialogView<TDialogResult>` extends it and adds `DialogResult`, `Show()`, `Close()`, and `Closing`/`Closed` events.
- `IMainView<TDialogResult, TDimension>` extends `IDialogView` and adds `Left`/`Top`/`Width`/`Height`.

Any view in the repository that participates in the framework ultimately derives from `IView` through one of those. It is a shared contract rather than an implementation: removing it breaks the interface hierarchy and therefore every view class (`UserForm` in `WinFormsApp1`, `UserView` in `WpfApp1`) and the module controllers that consume them.

## Relationships

- **Consumed by views:** `UserForm` (`WinFormsApp1/View/UserForm.cs`) implements `IMainView<DialogResult, int>`; `UserView` (`WpfApp1/View/UserView.xaml.cs`) implements `IMainView<bool?, double>`. Both are exported via MEF (`[Export(typeof(IMainView<...>))]`) and composed by each app's `DefaultModuleController`.
- **`DataContext` is not a pure contract member.** For the WPF side it is inherited from `Window`; for the WinForms side, `UserForm` *overrides* `object? DataContext` from `Control` and uses the setter as the hook where it wires up its `DataBindings` and event handlers via `SetupBindings`. So in practice, setting `DataContext` on a view is the framework's mechanism for handing the view model to the view and triggering its bindings.

## Behavior & Details

There is no behavior in this file — it is a single-member interface with no default implementations. The one thing worth knowing:

- `DataContext` is nullable and untyped. View implementations are expected to cast it to their specific view model type inside the setter (as `UserForm` does with `(IUserViewModel?)value`). Nothing in the framework validates the type at the contract level; a wrong type surfaces only inside the implementing view.

## Rough edges

- `IView`'s `DataContext` collides with framework members: `WinForms.Control` and WPF's `UIElement` already define `object? DataContext`/`object DataContext`. Implementations therefore inherit or override members with the same name and signature from different base classes; `UserForm` must explicitly `override` the WinForms version. This works today but makes the interface's contract partially determined by whichever UI framework's base type the implementer chooses, which is invisible to anyone reading `IView` alone.