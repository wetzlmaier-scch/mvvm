# `Mvvm/IMessageService.cs`

### Purpose

This file defines the `IMessageService` interface, the contract for showing modal message boxes (information, warning, error, and yes/no-style questions) from MVVM code. It exists so that view models can display user-facing dialogs without referencing a specific UI framework: the same call site works whether the app is hosted in WPF, Windows Forms, or a console.

### Role in the Codebase

This interface is the abstraction at the top of a small messaging subsystem. The core `Mvvm` namespace holds only the contract (this file plus the `MessageResult` enum in `Mvvm/MessageResult.cs`); the concrete behavior lives in separate projects, one per host UI: `Mvvm.Wpf/MessageService.cs`, `Mvvm.Winforms/MessageService.cs`, and `Mvvm.Console/MessageService.cs`. Each of those classes is marked `[Export(typeof(IMessageService))]`, so the composition container (MEF) injects the implementation that matches the host UI. If this interface were removed, none of the three implementations could exist and every view model using `[Import] public IMessageService MessageService { get; set; } = null!;` would stop compiling.

### Relationships

- **Implementations** (each exports this type to the MEF container): `MessageService` in `Mvvm.Wpf`, `MessageService` in `Mvvm.Winforms`, `MessageService` in `Mvvm.Console`.
- **Consumers**: view models obtain it via MEF composition, for example `AddUserViewModel` in `UserManagement/ViewModel/AddUserViewModel.cs`, which calls `MessageService.ShowError(...)` when validation fails.
- **Tests**: `TestProject1/UserViewModelTest.cs` substitutes a `Mock<IMessageService>` (Moq) for the real service.
- **`MessageResult`**: the return type of the three question methods, defined in `Mvvm/MessageResult.cs` as an enum mirroring Win32 message-box results (`OK`, `Cancel`, `Yes`, `No`, ...).

### Behavior & Details

The interface groups nine methods in three pairs-plus-one, all with the same shape: a fire-and-forget method and one or two question methods.

- **`ShowMessage(string message)` / `ShowMessage(string message, string caption)`** — informational dialog. The caption-less overload delegates to the two-argument one; implementations supply a default caption (e.g. `"Message"` in the WPF version).
- **`ShowWarning(...)`** — same pattern, warning icon; the single-argument overload uses a default caption such as `"Warning"`.
- **`ShowError(...)`** — same pattern, error icon; default caption `"Error"`.
- **`ShowYesNoCancelQuestion(string message, string caption)`** — returns a `MessageResult` telling the caller which button was pressed; the implementation preselects `Cancel` as the default button.
- **`ShowYesNoQuestion(string message, string caption)`** — `Yes`/`No` buttons, `No` preselected.
- **`ShowOkCancelQuestion(string message, string caption)`** — `OK`/`Cancel` buttons, `Cancel` preselected.

Two conventions are worth knowing before writing against this interface:

1. **The question methods always take both `message` and `caption`.** Unlike the notification methods, there are no caption-less overloads for questions, so call sites must supply both arguments.
2. **Question results are compared against `MessageResult` values, not `bool`.** For example a caller checking a yes/no answer tests for `MessageResult.Yes` explicitly; a `Cancel` or click-away is any other value. This matters because all three implementations cast the platform's native result to `MessageResult`, so the enum values line up with the user's actual button choice.

Because the interface is platform-neutral and mocked in unit tests, view models should be written against it only — never directly against `System.Windows.MessageBox` or `System.Windows.Forms.MessageBox`.

Rough edges: the `MessageResult` enum includes values the interface can never return for its three question methods (`None`, `Abort`, `Retry`, `Ignore`, `TryAgain`, `Continue`), since it mirrors the full Win32 set. That is harmless, but a reader should not expect any of those values from a call to this interface.