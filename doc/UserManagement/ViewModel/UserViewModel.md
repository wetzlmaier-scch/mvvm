# `UserManagement/ViewModel/UserViewModel.cs`

**File:** `UserManagement/ViewModel/UserViewModel.cs`

### Purpose

`UserViewModel` is the top-level view model for the UserManagement module and its MEF composition root. It exists as a single aggregation point so that anything that needs "the user management feature" can resolve one object (`IUserViewModel` or `IMainViewModel`) and reach both functional halves of the feature — adding a user and listing users — through it, without knowing which concrete classes implement them. It contains no business logic of its own; all the logic lives in the two child view models it imports.

### Role in the Codebase

It sits in `UserManagement/ViewModel/` and is the umbrella over that folder: the same folder holds the child pairs `IAddUserViewModel`/`AddUserViewModel` and `IShowUsersViewModel`/`ShowUsersViewModel`, and `UserViewModel` is the only class that ties them together. Its class is `partial`, but no other part of the class is defined in the repository; the designer-generated half (typical for MVVM templates) does not exist here.

The entry point depends on it directly: `ConsoleApp1/Program.cs` builds a MEF container and resolves `IUserViewModel` from it, throwing `IUserViewModel not found.` if the export is missing. In the WinForms host, `WinFormsApp1/View/UserForm.cs` receives the same object as its `DataContext` and unwraps it to `IUserViewModel` in `SetupBindings` to wire the controls. Removing this class breaks the export resolution in `Program.cs` and leaves `IUserViewModel` unimplemented, so both hosts lose the feature.

### Relationships

- `MainViewModelBase` (defined in `Mvvm/MainViewModelBase.cs`): inherited base class. It contributes the window lifecycle behaviour — `Icon` loading, `Shutdown(int)`, `CloseWindow()`, the `MainWindow` closing hook, and `IMainViewModel` implementation — none of which `UserViewModel` overrides. `UserViewModel` itself only adds the two imported child properties.
- `IUserViewModel` (defined in `UserManagement/ViewModel/IUserViewModel.cs`, same folder): the contract `UserViewModel` implements. It declares exactly the two properties below, both read-only on the interface.
- `IAddUserViewModel` / `IShowUsersViewModel` (same folder): the types of the imported properties. MEF satisfies them from `[Export]`ed implementations in the same folder — `AddUserViewModel` (`[Export(typeof(IAddUserViewModel))]`) and `ShowUsersViewModel` (`[Export(typeof(IShowUsersViewModel))]`). The class only references the interfaces, never the concrete types.
- Exported under two contracts: `[Export(typeof(IUserViewModel))]` and `[Export(typeof(IMainViewModel))]`. The second export is what lets the shell resolve it as the main window's view model, since `IMainViewModel` comes from the base class.

### Behavior & Details

The class body is entirely MEF wiring:

- `[Export]` attributes register `UserViewModel` in the composition container under both `IUserViewModel` and `IMainViewModel`.
- The two `[Import]` properties, `AddUserViewModel` and `ShowUsersViewModel`, are the only state. MEF sets them during composition from the matching exports in the same folder. They are initialized `= null!` so the compiler accepts the property, but any access before composition (or a container that lacks the matching export) is a null-reference hazard rather than a descriptive failure — see `Program.cs`, which does not null-check the children after resolving the container.
- There are no methods, commands, or state of its own. All user data (`FirstName`, `LastName`), validation, `SaveUserCommand`/`SaveUserAsync`, the `Users` collection, and loading logic live in the child view models; this class just hands them out, as `UserForm` does when it binds its text boxes and list to `viewModel.AddUserViewModel` and `viewModel.ShowUsersViewModel`.
- Window lifetime (`Shutdown`, icon, closing hooks) is inherited from `MainViewModelBase` and requires an `Application.Current` in a WPF/WinForms host; in the console host those inherited members are simply never called.

### Rough edges

- The `= null!` default on the imports silently defers failure: a missing export surfaces as a `NullReferenceException` at first use, not at composition time.
- The class is declared `partial`, but no second part of `UserViewModel` exists anywhere in the repository, so the keyword currently promises nothing.