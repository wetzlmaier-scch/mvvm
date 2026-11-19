### Purpose

`Mvvm/IDialogView.cs` defines `IDialogView<TDialogResult>`, the view-side contract for a top-level window in this repository's hand-rolled MVVM framework. It captures the minimum a window must expose so that a controller can open it, pass it a result, and react to it being dismissed — without knowing anything about WinForms or WPF.

### Role in the Codebase

The interface sits in the middle of the `Mvvm` project's view hierarchy: `IView` (in `Mvvm/IView.cs`) → `IDialogView` → `IMainView<TDialogResult, TDimension>` (in `Mvvm/IMainView.cs`). It is the generic "window" rung; `IMainView` is the specialization that adds geometry (`Left`, `Top`, `Width`, `Height`). Removing it would force `IMainView` to declare all the window members itself, so every caller typed against `IMainView` would keep working, but any code that needs just the dialog surface (`Show`, `Close`, `DialogResult`) with no positioning would have no interface to depend on.

### Relationships

- **Extends** `IView` (defined in `Mvvm/IView.cs`), which contributes the single `object? DataContext` property. That is how a controller binds a view model to the window.
- **Extended by** `IMainView<TDialogResult, TDimension>` (defined in `Mvvm/IMainView.cs`), which is the interface the controllers actually receive — e.g. as `IMainView<bool?, double>` in the WPF app and `IMainView<DialogResult, int>` in the WinForms app. `IDialogView` itself is the shared part of that contract.

### Behavior & Details

The members divide into three jobs:

- **Result channel**: `TDialogResult DialogResult { get; set; }`. The controller writes a result to the view before closing it; the view then uses it to decide how to behave on exit (the generic parameter lets each app type its result however it wants, e.g. `bool?` in the WPF app).
- **Lifecycle events**: `Closing` (a `CancelEventHandler`, so subscribers can cancel the close) and `Closed` (an `EventHandler`, firing after the window has gone). These mirror the standard framework close sequence and let the owning logic clean up or veto a close.
- **Open/close control**: `Show()` and `Close()` — the only way a controller drives the window's lifetime through this contract.

A typical controller flow is: set `DataContext`, set `DialogResult` when the answer is known, call `Close()`, and rely on `Closing`/`Closed` for teardown. The interface has no implementation; it is the shape each platform's main-window implementation is expected to satisfy.

Rough edges: The name `DialogView` (and the `DialogResult` member) suggests a modal dialog, but in this codebase it actually models the application's main window — the naming carries the "dialog result" pattern of modal windows into a non-modal top-level window.