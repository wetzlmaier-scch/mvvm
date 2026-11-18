### Purpose

`Mvvm.Wpf/MessageService.cs` is the WPF implementation of the `IMessageService` contract defined in the core `Mvvm` project. It exists so that WPF-based applications built on this MVVM framework can show message boxes without UI-framework types leaking into view models. View models depend only on `IMessageService`; at runtime, the WPF `MessageService` in this file is the concrete behavior behind that interface.

### Role in the Codebase

This is one of three parallel implementations of `IMessageService`, one per UI host: `Mvvm.Wpf/MessageService.cs` (WPF), `Mvvm.Winforms/MessageService.cs` (WinForms), and `Mvvm.Console/MessageService.cs` (console). Each is marked `[Export(typeof(IMessageService))]`, so MEF picks up the right one when the hosting application's assembly directory is scanned by `MvvmApplication`'s composition container. If this file were removed, a WPF host could no longer satisfy `IMessageService` imports — any WPF view model requesting the interface (see `UserManagement/ViewModel/AddUserViewModel.cs`, which takes `IMessageService` as a settable dependency) would fail to compose or be left unset.

### Relationships

- **`IMessageService`** (`Mvvm/IMessageService.cs`, namespace `Mvvm`): the interface this class implements. All nine methods map 1:1 to the interface's members; the interface is the source of truth for the contract.
- **`MessageResult`** (`Mvvm/MessageResult.cs`): the return type of the three question methods. It is a framework-neutral enum whose values are numbered to match WPF's `MessageBoxResult` (`OK = 1`, `Cancel = 2`, … `Yes = 6`, `No = 7`), which is what makes the casts in this class work.
- **MEF (`System.ComponentModel.Composition`)**: the `[Export(typeof(IMessageService))]` attribute is how this class registers itself with the `CompositionContainer` created in `MvvmApplication.OnStartup`.
- **Callers**: view models that import `IMessageService` (e.g. `AddUserViewModel`). Unit tests substitute a `Mock<IMessageService>` instead of this class (`TestProject1/UserViewModelTest.cs`), so tests never touch real message boxes.

### Behavior & Details

The class is a thin wrapper over WPF's `MessageBox.Show`, with no state and no other side effects. The methods fall into two groups:

- **Notification methods** — `ShowMessage`, `ShowWarning`, `ShowError`. Each shows an OK-only dialog. The icon differs by method: `MessageBoxImage.None` for messages, `Warning` for warnings, `Error` for errors. Each has a single-argument overload that supplies a fixed default caption: `"Message"`, `"Warning"`, or `"Error"` respectively.
- **Question methods** — `ShowYesNoCancelQuestion`, `ShowYesNoQuestion`, `ShowOkCancelQuestion`. Each shows a `Question`-icon dialog with its button set and returns the user's choice as a `MessageResult`. Only the two-argument form exists for questions; a caption is always required. The `defaultResult` passed to `MessageBox.Show` preselects a button: `Cancel` for Yes/No/Cancel, `No` for Yes/No, and `Cancel` for OK/Cancel — so pressing Enter (or Esc, where it maps to the default) yields that choice without an explicit click.

The non-obvious part is the return-value conversion: each question method casts WPF's `MessageBoxResult` straight to `MessageResult`, for example `(MessageResult)MessageBox.Show(..., MessageBoxResult.Cancel)`. That cast is only correct because the two enums were numbered identically. WPF's `MessageBoxResult` is `None = 0, OK = 1, Cancel = 2, Abort = 3, Retry = 4, Ignore = 5, Yes = 6, No = 7`, which is exactly the numbering `MessageResult` uses, so every value a WPF message box can return maps onto a same-named, same-numbered member.

### Rough edges

The `(MessageResult)MessageBoxResult` cast is safe today purely by numeric coincidence between the two enums. Nothing in the code enforces the alignment, and WPF's `MessageBoxResult` values are fixed by the framework — so the real risk is a future renumbering or addition in `MessageResult` silently corrupting results with no compile-time error. The WinForms sibling has the same kind of coupling.