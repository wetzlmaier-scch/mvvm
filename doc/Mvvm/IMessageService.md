# `Mvvm/IMessageService.cs`

### Purpose

This file defines the `IMessageService` interface, the contract for showing modal message boxes (information, warning, error, and yes/no-style questions) from MVVM code. It exists so that view models in framework-agnostic modules (like `UserManagement`) can display messages and ask the user questions without referencing WPF's `System.Windows.MessageBox` or WinForms' `System.Windows.Forms.MessageBox` types directly. It is the core piece that decouples business logic from the UI framework.

### Role in the Codebase

This interface sits at the top of a small messaging subsystem inside the `Mvvm` project. The `Mvvm` project acts as the core abstraction layer containing shared MVVM infrastructure (`ViewModelBase`, `IModuleController`, etc.) alongside this specific contract. By defining the interface here, the project guarantees that any view model can depend on user feedback mechanisms without creating a project-level dependency on `Mvvm.Wpf` or `Mvvm.Winforms`. If this file were removed, view models would lose their framework-agnostic way to interact with the user and be forced to import specific UI namespaces.

### Relationships

- **`MessageResult`**: This interface relies on the `MessageResult` enum (defined in `Mvvm/MessageResult.cs` in the same namespace) as the return type for its question methods. The enum maps the possible button clicks from a UI message box.
- **Implementations**: Three sibling projects provide concrete implementations decorated with `[Export(typeof(IMessageService))]` for MEF (Managed Extensibility Framework) discovery:
  - `Mvvm.Wpf/MessageService.cs` (WPF host)
  - `Mvvm.Winforms/MessageService.cs` (WinForms host)
  - `Mvvm.Console/MessageService.cs` (Console host)
- **Consumers**: View models in feature modules like `UserManagement` use MEF's `[Import]` attribute to pull this service in at runtime. For example, `UserManagement/ViewModel/AddUserViewModel.cs` contains `[Import] public IMessageService MessageService { get; set; } = null!;`.
- **Tests**: Unit tests, such as those in `TestProject1/UserViewModelTest.cs`, use Moq to mock this interface (`new Mock<IMessageService>()`) in order to test view model logic in isolation from the UI thread.

### Behavior & Details

This interface groups its nine methods into two categories based on whether they expect a response from the user:

- **One-way notifications**: 
  - `ShowMessage`, `ShowWarning`, and `ShowError` return `void`. Each provides an overload taking only a `message` string (which allows the implementation to supply a default caption like "Warning") and an overload taking both `message` and `caption`. Implementations are expected to display an informational box with a button matching the severity level.
- **User-input questions**: 
  - `ShowYesNoCancelQuestion`, `ShowYesNoQuestion`, and `ShowOkCancelQuestion` return a `MessageResult`. These methods are designed to map directly to the available buttons of the host platform's message box. 
  - *Notable edge case*: Notice that a standard `ShowYesNoQuestion` returns `MessageResult.No` as its fallback/default, while the other two questions default to `MessageResult.Cancel` (verified from `Mvvm.Wpf/MessageService.cs`). This subtle mismatch can be important when the view model uses the result to decide whether to proceed with an operation.

### Rough Edges

The `MessageResult` enum (which this interface depends on) contains values like `Abort`, `Retry`, `Ignore`, `TryAgain`, and `Continue` that are inherited from WinForms' `MessageBoxResult`. However, `IMessageService` only exposes methods for `YesNoCancel`, `YesNo`, and `OkCancel` questions. Consequently, these extra enum values can never be returned by a compliant implementation, making them dead weight in the enum specifically for the purpose of this interface.