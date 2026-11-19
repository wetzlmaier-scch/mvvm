### Purpose

`UserManagement/ViewModel/UserViewModel.cs` defines `UserViewModel`, the top-level view model for the UserManagement module. It exists as the single aggregation point for the feature: it holds the two screen-level view models — `IAddUserViewModel` and `IShowUsersViewModel` — so that hosts and views can depend on one object for "the whole user management feature" instead of resolving each piece separately.

### Role in the Codebase

It is the module's entry point within `UserManagement/ViewModel`. Sibling files define the child view models (`AddUserViewModel`, `ShowUsersViewModel`) and their interfaces; `UserViewModel` is what the rest of the solution actually resolves. It is declared `partial`, which leaves the door open for a generated or hand-written part to extend it, though no second part exists in this repository.

If it were removed, the feature becomes unresolvable: `ConsoleApp1/Program.cs` resolves `IUserViewModel` at startup and throws `"IUserViewModel not found."` when the export is missing, and the WinForms host wires its main form to it through `IMainViewModel`.

### Relationships

- **`MainViewModelBase`** (`Mvvm/MainViewModelBase.cs`): the inherited base. It brings in the main-window plumbing — `Icon`, `CloseWindow()`, `Shutdown(int exitCode)`, and a `Window.Closing` hook that routes through `OnClosing`/`OnClosed`. `UserViewModel` adds none of that itself; everything in it is aggregation.
- **`IUserViewModel`** (`UserManagement/ViewModel/IUserViewModel.cs`): the contract it implements. The interface declares the two properties read-only (`AddUserViewModel`, `ShowUsersViewModel`); the class implements the setters as MEF `[Import]` points, so the interface sees the values but never the assignment.
- **`IAddUserViewModel`** / **`IShowUsersViewModel`** (same folder): the two imported children, backed by `AddUserViewModel` and `ShowUsersViewModel` in the same directory.

MEF registers the class under two contracts: `[Export(typeof(IUserViewModel))]` for the feature entry point and `[Export(typeof(IMainViewModel))]` so the shell can treat it as the main window's view model.

Callers:

- `ConsoleApp1/Program.cs` resolves `IUserViewModel` from the composition container and drives the feature headlessly through `userViewModel.AddUserViewModel`.
- `WinFormsApp1/View/UserForm.cs` receives it via the `DataContext` setter and binds `txtFirstName`/`txtLastName` to `AddUserViewModel`, the user list to `ShowUsersViewModel.Users`, and the save button to `SaveUserCommand`.
- `UserManagement/Design/DesignUserViewModel.cs` derives from it for the designer, assigning design-time doubles directly through the settable properties — a use only possible because the class exposes setters while the interface does not.
- `TestProject1/UserViewModelTest.cs` instantiates it directly, bypassing MEF, by constructing `AddUserViewModel` in the object initializer.

### Behavior & Details

All behaviour lives in the composition. Each `[Import]` property is filled by the container from the matching `[Export]` of the child view models before the object is handed out:

```csharp
[Import]
public IAddUserViewModel AddUserViewModel { get; set; } = null!;

[Import]
public IShowUsersViewModel ShowUsersViewModel { get; set; } = null!;
```

The `null!` initializers acknowledge that the properties are null until MEF satisfies them. Consequence: anything holding a `UserViewModel` must treat both properties as populated only after the container has composed it. The test and the designer get around this by assigning the properties explicitly. There is no logic of its own — no commands, no state, no validation — so behavioural changes to the feature are made in the child view models, not here.

### Rough edges

- The `[Import]` setters are exposed on the class (required for MEF), while `IUserViewModel` hides them as read-only. Consumers depending on the interface cannot reassign the children, but anyone holding the concrete type can silently swap them, including outside of composition.
- Being `partial` with no second part in the repository suggests a generated or intended part that is not present.