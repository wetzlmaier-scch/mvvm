# Mvvm.Console/MessageService.cs

### Purpose
This file defines `MessageService`, the console-host implementation of `IMessageService` (defined in `Mvvm/IMessageService.cs`). It exists so that ViewModels in the library can display messages and warnings, and ask the user yes/no/ok-cancel questions, without knowing which UI technology is hosting the application. In a console app, the "display" part is coloured `Console.WriteLine` output; the question part is not implemented in a console host.

### Role in the Codebase
It is one of three sibling implementations of the same contract: `Mvvm.Console/MessageService.cs` (this file), `Mvvm.Wpf/MessageService.cs`, and `Mvvm.Winforms/MessageService.cs`. The `Mvvm` project defines only the interface, enum, and base ViewModel; each host-specific project contributes its own `MessageService`. The `Mvvm.Console` project contains nothing else but this service and its `.csproj`, so its entire reason to exist is to provide this export. If it were removed, any console application composing `IMessageService` from this assembly would fail to resolve the dependency (or would have to bring in the WPF/WinForms implementation instead).

### Relationships
- **`IMessageService`** (`Mvvm/IMessageService.cs`): the interface implemented here. All nine members, including the three question methods, are required by it.
- **`MessageResult`** (`Mvvm/MessageResult.cs`): the enum returned by the question methods.
- **MEF**: the `[Export(typeof(IMessageService))]` attribute (from `System.ComponentModel.Composition`) makes this class discoverable for dependency injection. Callers depend on the interface, not this class. For example, `UserManagement/ViewModel/AddUserViewModel.cs` declares `IMessageService MessageService` as a settable property and calls `MessageService.ShowError("User data is invalid!")`. Which concrete `MessageService` is supplied depends on which `Mvvm.*` assembly the host application composes.
- **Tests**: `TestProject1/UserViewModelTest.cs` does not use this class; it mocks `IMessageService` directly.

### Behavior & Details
All output is formatted as `{caption} | {message}`.

- `ShowMessage` prints to standard output in white; the single-argument overload delegates with the caption `"Message"`.
- `ShowWarning` prints to standard output in yellow; the single-argument overload uses the caption `"Warning"`.
- `ShowError` prints to standard **error** (`Console.Error`) in red — the only method that does this — with the caption `"Error"`.
- Each two-argument method sets `Console.ForegroundColor` and never restores the previous colour, so the console stays coloured as the last method that ran.
- `ShowYesNoCancelQuestion`, `ShowYesNoQuestion`, and `ShowOkCancelQuestion` each throw `NotImplementedException`. They exist only to satisfy the interface.

### Rough edges
- The question methods are unimplementable in a plain console host (no modal dialog), so they throw; any ViewModel calling a question method in a console app crashes at runtime rather than degrading gracefully.
- `ShowMessage` and `ShowWarning` permanently change the console foreground colour as a side effect of writing one line.