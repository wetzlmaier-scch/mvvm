### Purpose

The `Mvvm.Console` class library provides the console-based implementation of the `IMessageService` interface. It exists to allow framework-agnostic ViewModels to display user-facing messages in a terminal environment without depending on GUI frameworks like WPF or WinForms. This directory contains a single component that bridges the application’s message contract with `System.Console` output.

### Software Components

The entire logic of this directory resides in [MessageService.cs](Mvvm.Console/MessageService.md). This file defines the `MessageService` class, which is the sole public surface of the `Mvvm.Console` project.

`MessageService` implements the `IMessageService` contract from the `Mvvm` project. It handles three types of output:
*   **Informational messages**: Written to standard output in white.
*   **Warnings**: Written to standard output in yellow.
*   **Errors**: Written to standard error in red.

The class also includes stub implementations for interactive question methods (e.g., Yes/No, Ok/Cancel). These methods currently throw `NotImplementedException` because a non-GUI console environment cannot easily replicate modal dialog behavior.

### Relationships

`MessageService` depends on the `IMessageService` interface defined in `Mvvm/IMessageService.cs`. It uses the MEF (`System.ComponentModel.Composition`) attribute `[Export(typeof(IMessageService))]` to register itself with a `CompositionContainer`. This allows console host applications to inject the service into ViewModels via the `[Import]` attribute, ensuring that the ViewModels remain decoupled from the specific console implementation.

While other projects in the solution (such as WPF or WinForms clients) have their own implementations of `IMessageService`, `Mvvm.Console` is isolated to the console scenario. No other code within this directory depends on external files, and this directory does not depend on other directories beyond the `Mvvm` contract.

```mermaid
flowchart TD
    subgraph Mvvm.Console
        MS[MessageService]
    end
    subgraph Mvvm
        IMSG[IMessageService]
        MR[MessageResult]
    end
    
    MS --|implements| IMSG
    IMSG -.->|returns| MR
    MS --|uses| Console[System.Console]
```

### Rough edges

The interactive question methods (`ShowYesNoCancelQuestion`, `ShowYesNoQuestion`, `ShowOkCancelQuestion`) are unimplemented stubs that always throw `NotImplementedException`. Any code path in a console host that attempts to display an interactive prompt will crash.

## Contents
- [MessageService.cs](Mvvm.Console/MessageService.md)