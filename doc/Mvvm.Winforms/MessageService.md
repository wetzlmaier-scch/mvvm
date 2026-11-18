# Mvvm.Winforms/MessageService.cs

### Purpose

`Mvvm.Winforms/MessageService.cs` is the WinForms implementation of the cross-platform `IMessageService` interface. It wraps `System.Windows.Forms.MessageBox` so that view-models in the framework-agnostic `Mvvm` project can display messages, warnings, errors, and yes/no questions without ever touching WinForms types directly. The class is exported to MEF (`[Export(typeof(IMessageService))]`), so a container composes it into dependent view-models at runtime.

This is one of three interchangeable platform implementations of `IMessageService` in this repository — `Mvvm.Console/MessageService.cs` and `Mvvm.Wpf/MessageService.cs` do the same job for console and WPF hosts. A host application picks one by referencing the matching project.

### Role in the Codebase

The `Mvvm.Winforms` project is a thin adapter layer. It targets `net10.0-windows` with `UseWindowsForms` enabled and references only the `Mvvm` project. Its entire purpose is to satisfy the `IMessageService` contract with WinForms' `MessageBox`. If it were removed, WinForms applications consuming this library would have no composable `IMessageService` to display dialogs.

### Relationships

- **`Mvvm.IMessageService`** — the interface this class implements (defined in `Mvvm/IMessageService.cs`). All nine methods here are required by that contract.
- **`Mvvm.MessageResult`** — the enum returned by the question methods (defined in `Mvvm/MessageResult.cs`). Its values are deliberately numbered identically to WinForms' `MessageBoxButtons` enum, which is why the methods here can cast `MessageBox.Show`'s return value directly to `MessageResult`.
- **MEF consumers** — any view-model with an `[Import]` of `IMessageService` (e.g. `UserManagement/ViewModel/AddUserViewModel.cs`, which has an `IMessageService MessageService` property) receives this instance when the WinForms host composes the container.
- **`Mvvm.Console/MessageService.cs`**, **`Mvvm.Wpf/MessageService.cs`** — sibling implementations for other host platforms; not referenced by this project.

### Behavior & Details

- **Informational methods** (`ShowMessage`, `ShowWarning`, `ShowError`) each call `MessageBox.Show` with `MessageBoxButtons.OK` and the corresponding icon (`None`, `Warning`, `Error`). The single-argument overloads supply default captions: `"Message"`, `"Warning"`, `"Error"`.
- **Question methods** (`ShowYesNoCancelQuestion`, `ShowYesNoQuestion`, `ShowOkCancelQuestion`) display the respective button set with a `Question` icon and a default button (Button3 / Cancel for Yes-No-Cancel, Button2 / No or Cancel for the others). They cast the `MessageBoxButtons` result directly to `MessageResult`.
- The direct cast in the question methods relies on the numeric values of `MessageResult` matching `MessageBoxButtons` exactly. If either enum is ever renumbered, these casts silently return wrong results — there is no validation in this file.
- All methods are synchronous and block the UI thread while the dialog is open, as is standard for WinForms `MessageBox`.