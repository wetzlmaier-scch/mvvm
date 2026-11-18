### Purpose
`Mvvm/MessageResult.cs` defines `MessageResult`, a single platform-neutral enum that names the possible outcomes of a message-box question (OK, Cancel, Yes, No, and friends). It exists so the `Mvvm` core contract and all its UI-specific implementations can talk about user dialog answers without depending on WinForms, WPF, or the console.

### Role in the Codebase
This is a shared data type in the core `Mvvm` project (the `Mvvm.csproj` library), sitting right next to `IMessageService` in the same folder. It makes no decisions itself; it only supplies the return type for the three question methods. Removing it would break the signature of `IMessageService` and every implementation of that interface, so it is load-bearing for the whole messaging subsystem.

### Relationships
- `IMessageService` (`Mvvm/IMessageService.cs`) declares `MessageResult ShowYesNoCancelQuestion(...)`, `ShowYesNoQuestion(...)`, and `ShowOkCancelQuestion(...)`. Callers anywhere in the app that ask a question through that interface compare the returned value against these enum members.
- The implementations cast their platform-specific dialog results directly to this type:
  - `Mvvm.Winforms/MessageService.cs` — casts WinForms `MessageBox.Show` results (`MessageBoxButtons`), e.g. `(MessageResult)MessageBox.Show(...)`.
  - `Mvvm.Wpf/MessageService.cs` — casts WPF `MessageBoxResult` the same way.
  - `Mvvm.Console/MessageService.cs` — declares the question methods but throws `NotImplementedException`, so the console host never actually produces a result.

### Behavior & Details
The member numbering is the important part. `None` through `No` are assigned `0`–`7`, matching WinForms' `DialogResult`/`MessageBoxButtons` values exactly, and `TryAgain = 10` and `Continue = 11` match the corresponding WPF `MessageBoxResult` members (8 and 9 are unused in WPF, which is why there is a gap before 10).

That alignment is what makes the implementations' unchecked casts work: WinForms `MessageBoxButtons.YesNoCancel`/`YesNo`/`OKCancel` return 1, 2, 6, or 7, and WPF returns the same numbers for `OK`/`Cancel`/`Yes`/`No`, so a plain `(MessageResult)` cast yields the right member on both platforms without any mapping table.

Two consequences to know before touching this enum:

- The values are a contract, not just labels. Renumbering any member (or inserting a new one in the middle) silently corrupts the casts in `Mvvm.Winforms` and `Mvvm.Wpf` — the cast still compiles, but callers receive the wrong answer. Any change must keep the numbers aligned with both WinForms and WPF message-box enums.
- Several members are defined but unreachable: `None`, `Abort`, `Retry`, `Ignore`, `TryAgain`, and `Continue` can never be returned by the three `IMessageService` question methods, which only ever offer OK/Cancel, Yes/No, or Yes/No/Cancel button sets. They exist solely to preserve the numeric alignment with the framework enums.