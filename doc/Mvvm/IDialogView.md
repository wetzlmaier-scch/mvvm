# `Mvvm/IDialogView.cs`

### Purpose

This file defines `IDialogView<TDialogResult>`, the view-side contract for the application's main window in this repository's hand-rolled MVVM framework. It says what a top-level window can be asked to do (be shown, be closed, report a dialog result) and what it can announce (that it is closing, that it has closed), without committing to a UI framework. The generic `TDialogResult` exists because the two host apps disagree on what "the result of the window" is: WPF's `Window` has `Nullable<bool> DialogResult` while WinForms' `Form` has `DialogResult`.

### Role in the codebase

`IDialogView<TDialogResult>` is the middle rung of the view interface hierarchy in the `Mvvm` project. `IView` (defined in `Mvvm/IView.cs`) provides `object? DataContext`, and `IMainView<TDialogResult, TDimension>` (defined in `Mvvm/IMainView.cs`) extends this interface and adds `Left`/`Top`/`Width`/`Height`. No app ever implements this interface directly — both apps implement `IMainView`. Removing this file breaks the interface hierarchy and therefore `IMainView`, the two view classes (`UserForm`, `UserView`), and the module controllers that consume them.

### Relationships

- **Extends** `IView` from `Mvvm/IView.cs`, inheriting `object? DataContext { get; set; }`.
- **Extended by** `IMainView<TDialogResult, TDimension>` in `Mvvm/IMainView.cs`.
- **Implemented indirectly by** `WpfApp1/View/UserView.xaml.cs` (as `IMainView<bool?, double>`, where `bool?` mirrors WPF `Window.DialogResult`) and `WinFormsApp1/View/UserForm.cs` (as `IMainView<DialogResult, int>`, where `DialogResult` is the WinForms enum).
- **Consumed by** each app's `DefaultModuleController` via `IMainView`. In the WPF app, `Run()` calls `MainView.Show()` and `Shutdown()` calls `MainView.Close()`. In the WinForms app, `Run()` instead calls `Application.Run((Form)MainView)` directly (no call to the interface's `Show()`), and only `Shutdown()` calls `MainView.Close()`.

### Behavior & Details

No logic lives here — it is a pure interface contract with no default implementations. The members:

- `TDialogResult DialogResult { get; set; }` — the result of the window, typed by the implementing framework (e.g. `bool?` in WPF). In the WPF implementation it is inherited from `Window`.
- `event CancelEventHandler Closing` and `event EventHandler Closed` — announce the close sequence. `CancelEventHandler` (from `System.ComponentModel`, the only using in this file) signals the convention that a `Closing` subscriber can cancel the close.
- `void Show()` / `void Close()` — the lifecycle calls the module controllers make against the view.

One thing a reader cannot see from this file: the `TDialogResult` parameter is only as generic as it appears. `UserForm` types it as the WinForms `DialogResult` enum, so the "dialog result" of the WinForms app is, in fact, the `Form.DialogResult` property — the interface member is satisfied by an inherited framework member rather than by any code in the app itself. The interface imposes no constraint on `TDialogResult`; a wrong or unexpected type surfaces only in whatever code reads `DialogResult`.

### Rough edges

- `Show()` is effectively dead in one of the two host apps: the WinForms `DefaultModuleController.Run()` calls `Application.Run((Form)MainView)` instead of `MainView.Show()`, so the interface member is only exercised by the WPF app.
- `DialogResult` is written (`{ get; set; }`) but nothing in the repository reads it — the controllers never observe a `bool?` or `DialogResult` value after showing the window. The member is contract surface that current callers don't use.
- `Closing`/`Closed` have no subscribers in either host app today; they are declared but unlistened.