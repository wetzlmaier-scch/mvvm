### Purpose

The `Mvvm.Wpf` project is a WPF-specific support library that provides the application shell and UI messaging capabilities for module-based MVVM applications. It serves as the bridge between the framework-agnostic `Mvvm` core and the WPF runtime, handling application lifecycle management and modal dialog interactions. These files belong together because they are the two primary touchpoints a WPF consumer application has with the framework: one class hosts the application bootstrap, and the other handles user communication from view models.

### Software Components

The directory contains two main source code components, each with a distinct responsibility within the WPF host:

*   **Application Hosting:** The [MvvmApplication](Mvvm.Wpf/MvvmApplication.md) class acts as the base class for WPF `Application` objects. It is responsible for the entire bootstrap sequence, including scanning assemblies, building the MEF composition container, discovering modules, and managing the startup and shutdown lifecycle of those modules. It also installs global exception handlers to prevent crashes from unhandled exceptions on the UI thread or in background threads.
*   **UI Messaging:** The [MessageService](Mvvm.Wpf/MessageService.md) class implements the `IMessageService` contract. It encapsulates all interactions with `System.Windows.MessageBox`, allowing view models to display information, warnings, errors, and questions without directly referencing WPF types. This component is discovered via MEF and used by any view model that imports `IMessageService`.

### Relationships

`MvvmApplication` is the entry point that establishes the environment in which `MessageService` operates. During `OnStartup`, [MvvmApplication](Mvvm.Wpf/MvvmApplication.md) uses `CompositionHelper` to build a `CompositionContainer`. This container catalogs the WPF assemblies, which allows it to discover and import the `[Export]`ed `MessageService`. Consequently, `MessageService` does not reference `MvvmApplication`, but `MvvmApplication` depends on the successful composition of the environment that includes `MessageService`.

While `MessageService` does not interact directly with `MvvmApplication` at runtime, they share a dependency on the MEF composition container. `MvvmApplication` ensures the container is built and composed before modules run, which is the mechanism that makes `MessageService` available to view modules. If `MvvmApplication` fails to compose the container (e.g., due to a missing assembly or module load error), `MessageService` will not be available, and the application may shut down before any UI interaction occurs.

```mermaid
flowchart TD
    subgraph "Mvvm.Wpf"
        MvvmApp["MvvmApplication"]
        MsgSvc["MessageService"]
    end

    MEF["MEF Composition Container"]
    WPFUI["System.Windows.MessageBox"]

    MvvmApp -->|Builds & Composes| MEF
    MEF -->|Discovers via [Export]| MsgSvc
    MsgSvc -->|Delegates to| WPFUI
```

### Rough edges

*   The `MessageService` uses hardcoded German text for error dialogs in some contexts (specifically within `MvvmApplication`'s exception handling, which impacts the user experience of the host it serves).
*   `MvvmApplication`'s exception handling comments do not match its code: the `AppDomain.UnhandledException` handler is registered unconditionally, contrary to the comment suggesting it is skipped in Debug mode.

## Contents
- [MessageService.cs](Mvvm.Wpf/MessageService.md)
- [MvvmApplication.cs](Mvvm.Wpf/MvvmApplication.md)