### Purpose

`Mvvm/IMainView.cs` defines `IMainView<TDialogResult, TDimension>` — the contract for the application's main window in the shared `Mvvm` project. It exists so that the module controllers (`DefaultModuleController` in `WinFormsApp1` and `WpfApp1`) can drive the main window — show, close, set its data context — without knowing whether the host is a WinForms `Form` or a WPF `Window`.

It is a very small file: a single generic interface with four properties. Everything substantive about it comes from its generic parameters and the interface it extends.

### Role in the Codebase

This interface is the top of the view-side hierarchy in the `Mvvm` project:

- `IView` (`Mvvm/IView.cs`) declares only `DataContext`.
- `IDialogView<TDialogResult>` (`Mvvm/IDialogView.cs`) adds `DialogResult`, `Closing`/`Closed` events, `Show()`, and `Close()`.
- `IMainView<TDialogResult, TDimension>` adds the four geometry properties below.

The controller side of the main window is the sibling `IMainViewModel` (`Mvvm/IMainViewModel.cs`); the two go together — each host project's `DefaultModuleController` imports one `IMainView<...>` and one `IMainViewModel`, wires `MainView.DataContext = mainViewModel`, calls `MainView.Show()` in `Run()`, and `MainView.Close()` in `Shutdown()`.

### Relationships

- **Extends** `IDialogView<TDialogResult>` (`Mvvm/IDialogView.cs`), which in turn extends `IView`. Everything the controller needs for the main window (`Show`, `Close`, `DataContext`, `DialogResult`) is inherited; this interface only contributes geometry.
- **Implementors (MEF exports):**
  - `UserForm` (`WinFormsApp1/View/UserForm.cs`) — `[Export(typeof(IMainView<DialogResult, int>))]`, a `System.Windows.Forms.DialogResult`-typed dialog result with `int` pixel geometry. Its `Left`/`Top`/`Width`/`Height` come from the underlying `Form`.
  - `UserView` (`WpfApp1/View/UserView.xaml.cs`) — `[Export(typeof(IMainView<bool?, double>))]`, a WPF `Window` with nullable-bool result and `double` geometry.
- **Callers:** the two `DefaultModuleController` classes import the interface as a typed constructor argument. They only use the inherited members (`Show`, `Close`, `DataContext`); in this repository nothing reads or writes `Left`/`Top`/`Width`/`Height`.

### Behavior & Details

`TDimension` is a second generic parameter because the two UI frameworks disagree on the type of window geometry: WinForms uses `int` (device pixels), WPF uses `double` (device-independent units). The `TDimension` type therefore carries framework semantics without the interface knowing which host it is in.

The four properties — `Left`, `Top`, `Width`, `Height` — are plain get/set geometry. In the WinForms implementations they map directly onto `Form.Left`/`Top`/`Width`/`Height`; in WPF the analogous properties on `Window` are `Left`/`Top`/`Width`/`Height` too, which is what makes one interface cover both hosts.

### Rough edges

- The geometry properties are currently dead weight: no code in this repository reads or writes them. They exist for future positioning/resizing support.
- The `TDialogResult` parameter duplicates the one from `IDialogView<TDialogResult>` rather than being fixed or redeclared — harmless here, but it means `IMainView` carries a generic parameter it never mentions in its own members.