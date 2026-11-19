# `Mvvm/IMessageService.cs`

### Purpose

This file defines the `IMessageService` interface, the contract for showing message boxes (information, warning, error, and yes/no-style questions) from MVVM code. It exists so that view models in framework-agnostic projects can present user feedback without referencing a specific UI toolkit — WPF, WinForms, or console. The concrete host decides at runtime which `MessageBox`-like mechanism is used.

The interface has no base type; it is a standalone contract in namespace `Mvvm`.

### Role in the Codebase

This is a core abstraction in the `Mvvm` framework project — the shared, framework-agnostic layer that all host applications (`WpfApp1`, `WinFormsApp1`, `ConsoleApp1`, feature modules like `UserManagement`) build on. It sits alongside other framework primitives such as `IMainViewModel`, `IModuleController`, and the view-model base types.

The interface is the seam between "view model wants to tell the user something" and "some UI host actually draws it." Removing it would force every view model to take a direct dependency on a platform UI library, destroying the host-swap design of the framework.

### Relationships

- **`MessageResult`** (`Mvvm/MessageResult.cs`, same project): the enum returned by the three question methods. It is a platform-neutral stand-in for WPF's `MessageBoxResult` / WinForms' `DialogResult`, with values numbered to match them (`OK = 1`, `Cancel = 2`, `Yes = 6`, `No = 7`, ...).

- **Implementations** — three sibling class libraries each provide a `MessageService : IMessageService` decorated with `[Export(typeof(IMessageService))]` for MEF discovery:
  - `Mvvm.Wpf/MessageService.cs` — wraps WPF `MessageBox.Show`
  - `Mvvm.Winforms/MessageService.cs` — wraps WinForms `MessageBox.Show`
  - `Mvvm.Console/MessageService.cs` — writes to `Console.WriteLine` / `Console.Error`

- **Callers** — view models pull the service in via MEF `[Import]`. For example, `UserManagement/ViewModel/AddUserViewModel.cs` declares `[Import] public IMessageService MessageService { get; set; }` and calls `MessageService.ShowError("User data is invalid!")` when validation fails.

- **Tests** — `TestProject1/UserViewModelTest.cs` builds `new Mock<IMessageService>()` (Moq) and injects it, so view-model logic can be tested without a UI host.

Because callers depend only on the interface, they are agnostic about which of the three `MessageService` classes the MEF container hands them.

### Behavior & Details

The interface declares nine members in three families:

**Notification methods (no return value):**

- `ShowMessage(string message, string caption)` / `ShowMessage(string message)` — informational messages; the single-argument overload is a caption-less convenience.
- `ShowWarning(string message, string caption)` / `ShowWarning(string message)` — warnings.
- `ShowError(string message, string caption)` / `ShowError(string message)` — errors.

**Question methods (return `MessageResult`):**

- `ShowYesNoCancelQuestion(string message, string caption)`
- `ShowYesNoQuestion(string message, string caption)`
- `ShowOkCancelQuestion(string message, string caption)`

Each question method corresponds to a specific button set on the host platform's message box, and returns whichever button the user chose, expressed as the shared `MessageResult` enum. Callers typically branch on `MessageResult.Yes` / `MessageResult.No` / `MessageResult.Cancel`.

A few points worth knowing before you implement or extend this:

- **Default button conventions are an implementation detail, not part of this contract.** The interface does not say which button is preselected. In the WPF and WinForms implementations, `ShowYesNoQuestion` defaults to *No* while the other two questions default to *Cancel*. Any new implementation should match those conventions if it is to feel consistent across hosts.
- **Question methods are unsupported on the console host.** `Mvvm.Console/MessageService.cs` implements all three question methods as `throw new NotImplementedException()`. A view model that calls `ShowYesNoQuestion` will crash under the console host — this is an accepted limitation, but it constrains which view models can run headless.
- **The `MessageResult` enum is wider than the interface can produce.** Members like `None`, `Abort`, `Retry`, `Ignore`, `TryAgain`, and `Continue` mirror the full WinForms `DialogResult` set but are never returned by any of the nine methods. Do not write code that expects to receive those values from a question call.

### Rough edges

The `MessageResult` enum contains values (`None`, `Abort`, `Retry`, `Ignore`, `TryAgain`, `Continue`) that no implementation can ever return, since the interface only offers OK/Cancel, Yes/No, and Yes/No/Cancel button sets. The dead values exist only to keep the enum's numeric alignment with WPF's and WinForms' result enums intact, which the platform implementations rely on for their unchecked casts.