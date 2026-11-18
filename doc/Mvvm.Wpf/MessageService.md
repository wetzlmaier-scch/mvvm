# Mvvm.Wpf/MessageService.cs

### Purpose

`Mvvm.Wpf/MessageService.cs` defines the WPF implementation of `IMessageService` (declared in `Mvvm/IMessageService.cs`), a small abstraction over native message boxes. It exists so that view models can show informational, warning, error, and question dialogs without referencing WPF types. View models depend on the interface; this class is what actually displays the `System.Windows.MessageBox` at runtime.

### Role in the Codebase

This is a platform adapter inside the `Mvvm.Wpf` project. That project is the WPF host layer of an MEF-based modular application: `MvvmApplication.cs` in the same folder builds a `CompositionContainer` from assemblies at startup, and every exported type in the loaded assemblies becomes available for import. `MessageService` is marked `[Export(typeof(IMessageService))]`, so the container wires it into anything that carries an `[Import]` of `IMessageService` — for example the `MessageService` property in `UserManagement/ViewModel/AddUserViewModel.cs`, which calls `MessageService.ShowError(...)` when validation fails.

The `Mvvm.Wpf` namespace is not imported in this file, which is why this file doesn't see the interface — it comes from the `Mvvm` namespace via the project reference in `Mvvm.Wpf.csproj`. The same pattern exists for other UI stacks: `Mvvm.Console/MessageService.cs` and `Mvvm.Winforms/MessageService.cs` export equivalent implementations for their own platforms. A given application only loads the host assembly for the stack it runs on, so its container resolves to exactly one of these exports.

### Relationships

- **Implements** `IMessageService` (`Mvvm/IMessageService.cs`) — the nine members here are exactly the interface's members.
- **Returns** `MessageResult` (`Mvvm/MessageResult.cs`), a plain enum that mirrors WPF's `MessageBoxResult` values (`None`, `OK`, `Cancel`, `Abort`, `Retry`, `Ignore`, `Yes`, `No`, `TryAgain`, `Continue`), so callers never see a WPF type.
- **Consumed via MEF**: view models and other components pull it in with `[Import] public IMessageService MessageService { get; set; }`. Unit tests instead mock the interface directly (see `TestProject1/UserViewModelTest.cs`).
- Depends on `System.Windows.MessageBox` from WPF — this is the only place the WPF dependency shows up in the type's surface, which is the point of the abstraction.

### Behavior & Details

The class is a thin, stateless wrapper. Every method maps one-to-one onto a `MessageBox.Show` call with `MessageBoxButton.OK` for the informational family, and the appropriate button set for questions.

- **Informational methods** (`ShowMessage`, `ShowWarning`, `ShowError`) return `void`. Each has a two-argument form and a one-argument overload that supplies a default caption: `"Message"`, `"Warning"`, and `"Error"` respectively. Icons are `None`, `Warning`, and `Error`; button is always `OK`.
- **Question methods** return `MessageResult`. There are no caption overloads here — every question method requires an explicit caption.
  - `ShowYesNoCancelQuestion` — buttons `YesNoCancel`, default result `Cancel`.
  - `ShowYesNoQuestion` — buttons `YesNo`, default result `No` (pressed via the `E` key or as a default button behavior in WPF).
  - `ShowOkCancelQuestion` — buttons `OkCancel`, default result `Cancel`.

Each question method casts the raw WPF `MessageBoxResult` to the local `MessageResult` enum. Both enums use identical numeric values, so the cast is safe in practice, but it means the two enums must stay in lockstep — a value added to WPF's enum that isn't mirrored in `MessageResult` would be silently lost in the cast.

A subtle detail: WPF's `MessageBox.Show` returns `MessageBoxResult.None` if the box is dismissed by some path that doesn't map to a button (rare, but possible in some dialog interactions). In that case the question methods return `MessageResult.None`, not the declared default — the `Cancel`/`No` parameters are only the *default button*, not a fallback return value. Callers that treat "not affirmative" as the safe branch should check `!= MessageResult.OK` / `!= MessageResult.Yes` rather than assuming the default is always returned.

There is no logging, no async surface, and no customization point (font, owner window, modal-vs-modeless) — the class is deliberately opaque: it only answers "show this message, get back this result."

### Rough edges

- The `MessageBoxResult` → `MessageResult` cast couples the two enums silently; a future divergence in WPF's enum values would corrupt results without a compile-time error.
- Question methods lack the one-argument caption overloads their informational counterparts have, so callers writing `ShowYesNoQuestion("...", "…")` must always pass the caption — an asymmetry easy to trip over.