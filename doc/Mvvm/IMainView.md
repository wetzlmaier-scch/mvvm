# `Mvvm/IMainView.cs`

### Purpose

`IMainView<TDialogResult, TDimension>` is the shared contract for an application's main window in the `Mvvm` project. It exists so that a module controller can open, size, and close the main window without knowing whether the host is WinForms or WPF. The two type parameters keep the contract UI-framework-neutral: `TDialogResult` matches the host's native dialog-result type (WinForms `DialogResult`, WPF `bool?`), and `TDimension` matches its natural geometry type (`int` pixels for WinForms, `double` for WPF).

### Role in the Codebase

This interface is the top rung of the view interface hierarchy in `Mvvm/`. It sits in the shared project that both host applications reference, alongside its controller-side counterpart `IMainViewModel` (`Mvvm/IMainViewModel.cs`). Each host project's `DefaultModuleController` imports an `IMainView<...>` and drives the window lifecycle through it (`Show()`, `Close()`, `DataContext`). If it were removed, the controllers in both host projects would have nothing to inject, and the MEF exports on the main windows would lose their target type.

### Relationships

- **Extends** `IDialogView<TDialogResult>` (`Mvvm/IDialogView.cs`), which extends `IView` (`Mvvm/IView.cs`). From `IDialogView` it inherits `DialogResult`, the `Closing`/`Closed` events, `Show()` and `Close()`; from `IView` it inherits `DataContext`. This file itself adds only the four geometry properties below.
- **Implemented by** two MEF exports:
  - `UserForm` (`WinFormsApp1/View/UserForm.cs`) as `IMainView<DialogResult, int>` — its `Left`/`Top`/`Width`/`Height` come straight from `System.Windows.Forms.Form` (which uses `int`).
  - `UserView` (`WpfApp1/View/UserView.xaml.cs`) as `IMainView<bool?, double>` — a WPF `Window` whose geometry properties are `double`.
- **Consumed by** the two `DefaultModuleController` classes (`WinFormsApp1/Controller/DefaultModuleController.cs`, `WpfApp1/Controller/DefaultModuleController.cs`), which receive the instantiated interface via their `[ImportingConstructor]`, set `MainView.DataContext` to the view model, and call `Show()`/`Close()` in `Run()`/`Shutdown()`. The generic arguments at each call site must exactly match the corresponding `[Export]` attribute or MEF composition fails.

### Behavior & Details

The interface declares four get/set properties — `Left`, `Top`, `Width`, `Height` — all of type `TDimension`. They let controller code position and size the main window through the abstraction (e.g. restoring saved window placement) without touching a concrete `Form` or `Window`. Because the properties are settable, an implementer is expected to write straight through to the framework's own geometry members, as both current implementations do; there is no state of its own to keep in sync.

There is no logic in this file beyond the contract — it is a pure interface with no default members.