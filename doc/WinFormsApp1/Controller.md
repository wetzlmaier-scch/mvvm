### Purpose

The `Controller` directory serves as the entry point for the application's module lifecycle within the WinForms host. It contains the single implementation of the `IModuleController` interface, which bridges the host process (`Program.cs`) and the MVVM layers. This directory exists to satisfy MEF composition requirements, allowing the host to discover and execute the application's main module without explicit registration.

### Software Components

The directory contains one source file: `DefaultModuleController.cs`.

This file defines the `DefaultModuleController` class, the sole occupant of the directory. It implements the `IModuleController` contract from the shared `Mvvm` project. The class is responsible for:
*   Wiring the `UserForm` (view) and `UserViewModel` (view model) together.
*   Hosting the WinForms message loop via `Application.Run`.
*   Providing the lifecycle hooks (`Initialize`, `Run`, `Shutdown`) required by the MEF container.

As the only `IModuleController` in the project, it is the critical point where the host process transitions into the MVVM architecture.

### Relationships

The `DefaultModuleController` acts as a connector between the static host code and the export-based MVVM components.

*   **Upstream Caller:** `Program.cs` resolves `IModuleController` from the MEF container. It calls `Initialize()` followed by `Run()`. It does not resolve `Shutdown()`.
*   **Downstream Dependencies:** The constructor uses `[ImportingConstructor]` to inject `IMainView<DialogResult, int>` (resolved as `UserForm`) and `IMainViewModel` (resolved as `UserViewModel`).
*   **Wiring Mechanism:** The constructor assigns the view model to the view's `DataContext`. Because WinForms lacks automatic data binding, this assignment triggers `UserForm`'s internal `SetupBindings()` method, effectively initializing the UI logic.
*   **Execution Flow:** `Run()` blocks the thread by starting the WinForms message loop. `Shutdown()` is part of the contract but is not invoked by `Program.cs` in this specific host implementation.

```mermaid
flowchart TD
    subgraph "WinFormsApp1 Host"
        P[Program.cs]
    end
    
    subgraph "Controller Directory"
        DMC[DefaultModuleController]
    end
    
    subgraph "MVVM / View Layer"
        UF[UserForm]
        UVM[UserViewModel]
    end
    
    P -->|Resolves & Calls| DMC
    DMC -->|Injects & Wires| UF
    DMC -->|Injects & Wires| UVM
    DMC -->|Blocks on Run()| UF
```

### Rough edges

*   `Shutdown()` is effectively dead code in this host; `Program.cs` never calls it, so the lifecycle contract's teardown mechanism is unused.
*   The controller does not invoke `CloseWindow()` or `Shutdown(int exitCode)` on the `IMainViewModel`, meaning the view model cannot signal an exit code back to the host through these intended hooks.

## Contents
- [DefaultModuleController.cs](WinFormsApp1/Controller/DefaultModuleController.md)