# `Mvvm.Console/MessageService.cs`

This file defines `MessageService`, the console-host implementation of the `IMessageService` interface.

### Purpose

`MessageService` exists so that framework-agnostic ViewModels can display user-facing messages to a console window. It provides the console variant of the `IMessageService` contract (declared in `Mvvm/IMessageService.cs`), which otherwise has parallel WPF and WinForms implementations. Its job is to write colored, captioned lines to `System.Console` for informational, warning, and error messages.

### Role in the Codebase

This is the sole source file in the `Mvvm.Console` class library, making it the entire public surface of that project. It is discovered at composition time by MEF: the class carries `[Export(typeof(IMessageService))]` (from `System.ComponentModel.Composition`), so a `CompositionContainer` in a console host project can inject an instance into any component that declares `[Import] IMessageService`.

View models — for example, `AddUserViewModel` in `UserManagement/ViewModel/AddUserViewModel.cs` — hold an `[Import]`-marked `IMessageService MessageService` property and call methods like `ShowError("User data is invalid!")` to report validation failures to the user. Unit tests in `TestProject1/UserViewModelTest.cs` substitute a `Mock<IMessageService>` instead, so they never reach this concrete class.

### Relationships

- **`IMessageService`** (`Mvvm/IMessageService.cs`, namespace `Mvvm`): the contract implemented here. All nine members in the class map 1:1 to the interface's methods; none are added and none are omitted.
- **`MessageResult`** (defined in the `Mvvm` project): the return type of the three `Show*Question` methods.
- **MEF**: `[Export(typeof(IMessageService))]` is the registration mechanism. Callers depend on the interface only, never on this class by name.

### Behavior & Details

All nine members mirror the interface:

- **`ShowMessage(string message, string caption)`** — sets the console foreground color to `White` and writes `"{caption} | {message}"` to standard output via `Console.WriteLine`.
- **`ShowMessage(string message)`** — a convenience overload that delegates to the two-argument version with the fixed caption `"Message"`.
- **`ShowWarning(string message, string caption)`** — same as `ShowMessage` but uses `ConsoleColor.Yellow`.
- **`ShowWarning(string message)`** — delegates with the fixed caption `"Warning"`.
- **`ShowError(string message, string caption)`** — uses `ConsoleColor.Red` and writes to **standard error** via `Console.Error.WriteLine` rather than standard output. This is the only one of the three message families that routes to stderr.
- **`ShowError(string message)`** — delegates with the fixed caption `"Error"`.
- **`ShowYesNoCancelQuestion`, `ShowYesNoQuestion`, `ShowOkCancelQuestion`** — all three throw `NotImplementedException` unconditionally. They are implemented only to satisfy the interface.

### Rough edges

The three question methods (`ShowYesNoCancelQuestion`, `ShowYesNoQuestion`, `ShowOkCancelQuestion`) are stubs that always throw `NotImplementedException`. Any code path that reaches a question-style prompt in a console host will crash. This mirrors what the WPF/WinForms implementations offer (interactive modal dialogs), but the console variant simply cannot provide that functionality, and no alternative (e.g. reading a keystroke) has been implemented.