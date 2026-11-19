### Purpose
`Mvvm.Wpf/MessageService.cs` defines `MessageService`, the WPF implementation of the `IMessageService` contract. It exists so that framework-agnostic view models can show modal dialogs (info, warning, error, and yes/no/ok-cancel questions) without referencing WPF themselves — this class is the only place in that path that touches `System.Windows.MessageBox`.

### Role in the Codebase
This is one of three parallel `IMessageService` implementations, one per UI host: WPF (this file), WinForms (`Mvvm.Winforms/MessageService.cs`), and console (`Mvvm.Console/MessageService.cs`). Each lives in its own host-support project, and exactly one is loaded into any given application's MEF catalog.

The class is registered for MEF discovery via `[Export(typeof(IMessageService))]`. At startup, `Mvvm.Wpf/MvvmApplication.cs` builds a `CompositionContainer` (through `CompositionHelper` in `Common/Composition/CompositionHelper.cs`) that catalogues the WPF host's assemblies and thus finds this export. If the file were removed, any WPF host that imports `IMessageService` would fail composition, because there would be no export to satisfy it.

### Relationships
- **`IMessageService`** (`Mvvm/IMessageService.cs`, namespace `Mvvm`): the interface this class implements. All nine members map 1:1 to the interface; nothing is added and nothing is omitted.
- **`MessageResult`** (`Mvvm/MessageResult.cs`, namespace `Mvvm`): the return type of the three question methods. The numeric values are deliberately aligned with WPF's `MessageBoxResult` enum (OK = 1, Cancel = 2, Yes = 6, No = 7), which is why the cast in each question method is safe.
- **MEF (`System.ComponentModel.Composition`)**: the `[Export]` attribute is how this class becomes discoverable; callers never name `MessageService` directly.
- **WPF's `MessageBox`** (`System.Windows`): the sole UI dependency; every method delegates to `MessageBox.Show`.
- **Callers**: view models that import `IMessageService` — for example `UserManagement/ViewModel/AddUserViewModel.cs` has an `IMessageService MessageService` property. Unit tests substitute a `Mock<IMessageService>` (`TestProject1/UserViewModelTest.cs`) instead of this class, so tests never exercise it.

### Behavior & Details
Every method is a thin, blocking call to `MessageBox.Show`. Since the dialogs are modal, each call suspends the UI thread until the user dismisses the box.

**Notifications (no return value)** — `ShowMessage`, `ShowWarning`, and `ShowError` each come in two overloads:

- The two-argument form shows a `MessageBoxButton.OK` box with the given caption and an image of `None`, `Warning`, or `Error` respectively.
- The one-argument form just supplies a default caption — `"Message"`, `"Warning"`, or `"Error"` — and delegates to the two-argument form.

**Questions (return `MessageResult`)** — the three question methods each pass a `MessageBoxResult` value as `MessageBox.Show`'s `buttonResult` parameter. That parameter determines what the box returns when dismissed without a button press (e.g., Alt+F4), so each method has a deliberate "escape" default:

- `ShowYesNoCancelQuestion` → buttons `YesNoCancel`, default result `Cancel`.
- `ShowYesNoQuestion` → buttons `YesNo`, default result `No`.
- `ShowOkCancelQuestion` → buttons `OkCancel`, default result `Cancel`.

The result is cast from WPF's `MessageBoxResult` to the framework's `MessageResult`. Because the two enums share numeric values, the cast preserves meaning for the results that can actually occur (`OK`, `Cancel`, `Yes`, `No`).

Callers branch on the returned `MessageResult` (e.g., proceed only on `MessageResult.Yes`). A caller should also be ready for the "escape" default if the dialog is closed by means other than a button.

### Rough edges
The `MessageResult` enum contains values that none of the question methods can return (`None`, `Abort`, `Retry`, `Ignore`, `TryAgain`, `Continue`); they exist only because the enum mirrors `MessageBoxResult` wholesale.