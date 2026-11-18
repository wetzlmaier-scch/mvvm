# `Mvvm/MessageResult.cs`

### Purpose

This file defines a single enum, `MessageResult` — a platform-neutral type that represents the outcome of a yes/no-style question shown to the user. It exists so that the abstractions in the `Mvvm` core project can specify a question's return type without depending on any GUI framework, whose own result enums (`MessageBoxResult` in WPF, `DialogResult` in WinForms) differ from each other.

### Role in the Codebase

It is a small shared data type in the core `Mvvm` project, alongside the `IMessageService` interface that consumes it. It makes no decisions itself; it only names the possible outcomes of a question. If it were removed, `IMessageService`'s three `Show...Question` methods and every `MessageService` implementation could no longer compile, because they use `MessageResult` as their return type.

### Relationships

- **`IMessageService`** (same folder, `Mvvm/IMessageService.cs`) declares `MessageResult ShowYesNoCancelQuestion(...)`, `ShowYesNoQuestion(...)`, and `ShowOkCancelQuestion(...)` as its question-returning methods. Callers of that interface use this enum to branch on the user's answer.
- **Implementations** in the satellite projects — `Mvvm.Winforms/MessageService.cs`, `Mvvm.Wpf/MessageService.cs`, and `Mvvm.Console/MessageService.cs` — cast their platform-specific dialog results to `MessageResult` and return them. The console implementation throws `NotImplementedException` for all three question methods, so this enum is only meaningful in GUI contexts.

### Behavior & Details

The members mirror the WinForms `DialogResult` values 0–7 (`None`, `OK`, `Cancel`, `Abort`, `Retry`, `Ignore`, `Yes`, `No`) plus `TryAgain = 10` and `Continue = 11`, which match the numeric values of `System.Windows.Forms.DialogResult`. That alignment is deliberate: the WinForms implementation simply does an unchecked cast `(MessageResult)MessageBox.Show(...)` and gets a correctly-named member for every `DialogResult` the message boxes can return.

The WPF implementation casts `MessageBoxResult` values (`No`, `Cancel`, `OK`) into this enum; those three values exist here with matching numeric values, so the cast is safe.

Practically, callers only ever observe the subset returned by the interface's three methods: `Yes`/`No`/`Cancel` from `ShowYesNoCancelQuestion`, `Yes`/`No` from `ShowYesNoQuestion`, and `OK`/`Cancel` from `ShowOkCancelQuestion`. The remaining members (`None`, `Abort`, `Retry`, `Ignore`, `TryAgain`, `Continue`) exist only to keep the numeric values cast-compatible with `DialogResult`; no method in this codebase can currently produce them.

### Rough edges

The WinForms implementations rely on an unchecked cast between a WinForms enum and this one. Any `DialogResult` member with a numeric value that is not a member of `MessageResult` would round-trip as a silently "wrong" value, and the cast breaks if either enum's values ever change. Today the values line up, but nothing in the code enforces that.