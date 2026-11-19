### Purpose

`Mvvm/MessageResult.cs` defines `MessageResult`, a single platform-neutral enum that names the possible outcomes of a message-box question. It exists so the `Mvvm` core project — which has no dependency on WPF or WinForms — can specify what an `IMessageService` question method returns, while each platform satellite project does the actual casting from its native result type.

### Role in the Codebase

The `Mvvm` project is the shared, framework-agnostic core: interfaces and base types that the satellite adapters (`Mvvm.Wpf`, `Mvvm.Winforms`, `Mvvm.Console`) implement. `MessageResult` sits alongside `IMessageService` as its companion value type. Without it the three `Show*Question` methods on `IMessageService` would have no return type to speak of, and the WPF/WinForms adapters could not share a contract with callers.

### Relationships

- **`IMessageService`** (`Mvvm/IMessageService.cs`, same namespace) declares `MessageResult ShowYesNoCancelQuestion(...)`, `ShowYesNoQuestion(...)`, and `ShowOkCancelQuestion(...)`. Any view model that asks the user a yes/no-style question through this interface receives a `MessageResult` and switches on it — that is the primary consumer pattern.
- **`Mvvm.Wpf/MessageService.cs`** casts WPF's `MessageBoxResult` straight to `MessageResult`: `(MessageResult)MessageBox.Show(message, caption, MessageBoxButton.YesNoCancel, MessageBoxImage.Question, MessageBoxResult.Cancel)`.
- **`Mvvm.Winforms/MessageService.cs`** casts WinForms' `DialogResult` the same way: `(MessageResult)MessageBox.Show(message, caption, MessageBoxButtons.YesNo, MessageBoxIcon.Question, MessageBoxDefaultButton.Button2)`.
- **`Mvvm.Console/MessageService.cs`** never produces a `MessageResult` — its question methods throw `NotImplementedException`.

### Behavior & Details

The enum members are deliberately numbered to mirror both WPF's `MessageBoxResult` and WinForms' `DialogResult`:

- `None = 0`, `OK = 1`, `Cancel = 2`, `Abort = 3`, `Retry = 4`, `Ignore = 5`, `Yes = 6`, `No = 7`
- Then a gap: `TryAgain = 10` and `Continue = 11`

`8` and `9` are absent because both framework enums leave them unused; `MessageResult` preserves that hole rather than renumbering. This numeric alignment is exactly what makes the `(MessageResult)` casts in the WPF and WinForms `MessageService` implementations safe today — nothing in the compiler enforces the correspondence.

Only a small subset is actually reachable through `IMessageService`: `OK`, `Cancel`, `Yes`, and `No` are the outcomes its button sets can return. The rest (`Abort`, `Retry`, `Ignore`, `TryAgain`, `Continue`) are carried for parity with the platform enums but no `IMessageService` method can produce them.

### Rough edges

The cast safety in the WPF/WinForms adapters depends entirely on this enum's values staying numerically locked to two third-party enums. A future renumbering or value addition here would silently corrupt the adapters' unchecked casts with no compile-time warning.