# `Mvvm.Winforms/MessageService.cs`

### Purpose

This file defines `MessageService`, the WinForms implementation of the `IMessageService` interface (declared in `Mvvm/IMessageService.cs`). It exists so that view models — which live in framework-agnostic projects and never touch a UI framework directly — can show modal message boxes by calling through the interface. Each of the nine interface methods is a thin wrapper around `System.Windows.Forms.MessageBox.Show`, choosing the appropriate buttons and icon.

### Role in the Codebase

The `Mvvm.Winforms` project is a thin adapter: this single class is its entire functional surface, and it is the WinForms host's counterpart to two sibling implementations, `Mvvm.Wpf/MessageService.cs` and `Mvvm.Console/MessageService.cs`. Each is decorated with `[Export(typeof(IMessageService))]` (MEF, from `System.ComponentModel.Composition`) so a `CompositionContainer` can discover and supply it at runtime. Exactly which implementation gets composed depends on which host project loads which assemblies.

If this class were removed, a WinForms host of this application would have no concrete `IMessageService` export to satisfy view model imports, and any attempt to show a message or question from a view model would fail at composition or runtime.

### Relationships

- **`IMessageService`** (`Mvvm/IMessageService.cs`, namespace `Mvvm`): the contract implemented here. All nine methods map 1:1 to the interface's members; no members are added.
- **`MessageResult`** (`Mvvm/MessageResult.cs`): the enum returned by the three question methods. The `MessageBoxResult` value from `MessageBox.Show` is cast straight to `MessageResult`; the two enums have deliberately aligned numeric values (e.g. `OK = 1`, `Yes = 6`, `No = 7`), so the cast is valid.
- **`System.Windows.Forms.MessageBox`**: the only WinForms dependency; every method delegates to it.
- **Callers**: view models that import `IMessageService` are the real consumers. For example, `UserManagement/ViewModel/AddUserViewModel.cs` has an `IMessageService MessageService` property and calls `MessageService.ShowError("User data is invalid!")`. Unit tests substitute a `Mock<IMessageService>` instead of this class (`TestProject1/UserViewModelTest.cs`), so tests never exercise this implementation.

### Behavior & Details

- **Notify-only methods** — `ShowMessage`, `ShowWarning`, `ShowError`: each shows a modal box with a single OK button and the matching icon (`None`, `Warning`, `Error`). Each has an overload taking just a message that delegates to the two-argument version with a default caption: `"Message"`, `"Warning"`, or `"Error"` respectively.
- **Question methods** — `ShowYesNoCancelQuestion`, `ShowYesNoQuestion`, `ShowOkCancelQuestion`: each returns a `MessageResult` cast from the `MessageBoxResult`. The default button is set so the button associated with `No`/`Cancel` is the default (Button2 or Button3), meaning pressing Enter declines rather than confirms.
- All dialogs are modal, so the calling view model blocks until the user dismisses the box.

### Rough edges

- The direct `(MessageResult)` cast relies entirely on the two enums keeping their numeric values in lockstep. There is no shared definition; if either file is edited independently, a mismatch would silently produce wrong results rather than a compile error.
- The `MessageResult` values `Abort`, `Retry`, `Ignore`, `TryAgain`, and `Continue` can never be returned by this class, since no method here offers those buttons — the cast only ever yields `OK`, `Cancel`, `Yes`, or `No`.