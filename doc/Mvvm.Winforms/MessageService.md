# `Mvvm.Winforms/MessageService.cs`

### Purpose

This file defines `MessageService`, the WinForms implementation of `IMessageService` (declared in `Mvvm/IMessageService.cs`). It exists so that view models — which live in framework-agnostic projects and must not reference WinForms types directly — can display modal message boxes and ask yes/no-style questions. Each method is a thin wrapper around `System.Windows.Forms.MessageBox`, translating the shared interface contract into a native WinForms dialog.

It is one of three interchangeable platform implementations of the same contract: `Mvvm.Console/MessageService.cs` and `Mvvm.Wpf/MessageService.cs` do the identical job for console and WPF hosts. Which one a running app actually uses is decided at runtime by MEF discovery, not at compile time.

### Role in the Codebase

The `Mvvm.Winforms` project is a thin adapter layer whose entire source content is this one class; its `.csproj` targets `net10.0-windows` with Windows Forms enabled and references only the `Mvvm` project. `MessageService` is the leaf node of the messaging subsystem: the `Mvvm` project holds the contract and the `MessageResult` enum, and the satellite projects supply the platform behaviour.

The class carries `[Export(typeof(IMessageService))]` (from `System.ComponentModel.Composition`), which registers it as a MEF export. At startup, `WinFormsApp1/Program.cs` builds a `CompositionContainer` via `Common.Composition.CompositionHelper.CreateCompositionContainer()`, which scans assemblies on disk for such exports; any part with an `[Import]` of `IMessageService` then receives this class. Remove this file (or its export attribute) and the WinForms host's container has no `IMessageService` to satisfy.

### Relationships

- **`Mvvm.IMessageService`** — the interface implemented here; all nine methods are exactly its members.
- **`Mvvm.MessageResult`** — the return type of the three question methods, defined in `Mvvm/MessageResult.cs`.
- **`System.Windows.Forms.MessageBox`** — the WinForms API every method delegates to.
- **MEF consumers** — view models that import `IMessageService` are the real callers. For example, `UserManagement/ViewModel/AddUserViewModel.cs` has an `IMessageService MessageService` property and calls `MessageService.ShowError("User data is invalid!")`. Unit tests bypass this class entirely and mock the interface directly (`TestProject1/UserViewModelTest.cs` uses a Moq `Mock<IMessageService>`).

### Behavior & Details

The nine methods come in three families:

- **Information (`ShowMessage`)** — `MessageBoxIcon.None`, OK button only. The single-argument overload delegates to the two-argument one with a default caption of `"Message"`.
- **Warning (`ShowWarning`)** — `MessageBoxIcon.Warning`, OK button. Default caption `"Warning"`.
- **Error (`ShowError`)** — `MessageBoxIcon.Error`, OK button. Default caption `"Error"`.

The single-argument overloads exist only for `ShowMessage`, `ShowWarning`, and `ShowError`; the question methods all require an explicit caption (there are no default-caption overloads for them).

- **Question methods** — each shows a question-icon dialog with a distinct button set and a preselected default button, and returns the user's choice as a `MessageResult`:
  - `ShowYesNoCancelQuestion`: Yes/No/Cancel, default button is Cancel.
  - `ShowYesNoQuestion`: Yes/No, default button is No.
  - `ShowOkCancelQuestion`: OK/Cancel, default button is Cancel.

All three return via an unchecked cast, e.g. `(MessageResult)MessageBox.Show(message, caption, MessageBoxButtons.YesNo, MessageBoxIcon.Question, MessageBoxDefaultButton.Button2)`. The cast works because `MessageResult`'s numeric values mirror WinForms' `DialogResult` values (`OK = 1`, `Cancel = 2`, `Yes = 6`, `No = 7`). Callers should compare the result against a specific `MessageResult` value (e.g. `MessageResult.Yes`) rather than treating it as a boolean; any other button press — including Cancel — means the affirmative was not chosen.

### Rough edges

The direct `(MessageResult)` casts couple this class to the numeric alignment of two independently defined enums. If `MessageResult` were ever renumbered — or a `DialogResult` value that `MessageResult` does not define were returned — the cast would silently produce a wrong result with no compile-time or runtime error.