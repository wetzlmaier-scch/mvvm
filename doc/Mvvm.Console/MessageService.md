# `Mvvm.Console/MessageService.cs`

This file defines `MessageService`, the console-host implementation of the `IMessageService` interface (declared in `Mvvm/IMessageService.cs`). It exists so that ViewModels in the framework-agnostic `Mvvm` project can display messages, warnings, and errors in a console application without referencing any WPF or WinForms types.

### Purpose
The `Mvvm` project defines a messaging contract that its ViewModels use to surface information to the user. Each host UI technology provides its own implementation of that contract: `Mvvm.Wpf` and `Mvvm.Winforms` use native message boxes, and this class covers the console scenario, where the only available output is colored text on the terminal.

### Role in the Codebase
This is the single source file of the `Mvvm.Console` class library. It is decorated with `[Export(typeof(IMessageService))]`, so MEF discovers it at composition time and supplies it to any component that imports `IMessageService`. `Mvvm.Console.csproj` references only the `Mvvm` project (for the interface and `MessageResult`) and the `System.ComponentModel.Composition` package, keeping the console adapter free of any UI-framework dependency.

A caller in this context is a ViewModel injected with the interface — for example `UserManagement/ViewModel/AddUserViewModel.cs` has an `IMessageService MessageService` property — or a unit test that instead mocks `IMessageService` directly (`TestProject1/UserViewModelTest.cs`).

### Relationships
- **`IMessageService`** (`Mvvm/IMessageService.cs`): the contract implemented here. All nine members are required by it; none are added.
- **`MessageResult`** (`Mvvm/MessageResult.cs`): the enum returned by the three question methods.
- **MEF**: `[Export(typeof(IMessageService))]` (from `System.ComponentModel.Composition`) makes this class discoverable for dependency injection. Callers depend on the interface, not this class.

### Behavior & Details
The six display methods all follow the same shape: set `System.Console.ForegroundColor`, then write a line of the form `{caption} | {message}`.

- `ShowMessage` writes in white to standard output; the single-argument overload delegates with the default caption `"Message"`.
- `ShowWarning` writes in yellow to standard output; the single-argument overload defaults the caption to `"Warning"`.
- `ShowError` writes in red, but to `Console.Error` rather than `Console.Out`; the single-argument overload defaults the caption to `"Error"`.

There is no state: each method sets the foreground color, writes one line, and leaves the color as set. Nothing here restores the previous console color, so subsequent unrelated console output inherits the color used by the last message.

The three question methods — `ShowYesNoCancelQuestion`, `ShowYesNoQuestion`, `ShowOkCancelQuestion` — exist only to satisfy the interface and each body is a single `throw new NotImplementedException()`. A console host cannot present a modal button dialog, so calling any of them from a console-hosted ViewModel crashes the application. The other two implementations (`Mvvm.Wpf`, `Mvvm.Winforms`) return a real `MessageResult` from a native dialog; this one cannot.

### Rough edges
- The question methods throw at runtime instead of degrading to a text prompt, so any ViewModel path that reaches a question is untestable in the console host.
- `Console.ForegroundColor` is a process-wide setting that is never restored after a message, which can leak into any other console output in the host application.