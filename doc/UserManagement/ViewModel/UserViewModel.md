# UserViewModel.cs

`UserManagement/ViewModel/UserViewModel.cs`

### Purpose

`UserViewModel` is the top-level (main) view model for the user-management feature. It exists to compose the feature's two child view models — adding a user and listing users — under one object, and to serve as the composition root's single exported entry point for the whole feature. It contains no logic of its own; it is a shell that wires the pieces together.

### Role in the Codebase

It is the root node of the `UserManagement` module's view-model hierarchy. `IUserViewModel` (declared in `UserManagement/ViewModel/IUserViewModel.cs`) exposes exactly the two children, `AddUserViewModel` and `ShowUsersViewModel`, and `UserViewModel` is its only production implementation.

Because it is the aggregate, it is the type that hosts and DI consumers ask for:

- `ConsoleApp1/Program.cs` resolves `IUserViewModel` from the MEF container and drives the feature through `userViewModel.AddUserViewModel`.
- `WinFormsApp1/View/UserForm.cs` takes an `IUserViewModel` as its data context and binds directly to `AddUserViewModel`'s properties and commands.
- `WpfApp1/View/UserView.xaml` binds the same way (`AddUserViewModel.FirstName`, `AddUserViewModel.SaveUserCommand`, etc.), with the design-time instance supplied by `DesignUserViewModel`.
- `TestProject1/UserViewModelTest.cs` constructs a `UserViewModel` directly, injecting a real `AddUserViewModel`, to test save behaviour end-to-end through the child.

Remove it and the feature's container export, the WinForms/WPF root bindings, and the console entry point all lose their target.

### Relationships

- Inherits `MainViewModelBase` (in `Mvvm/MainViewModelBase.cs`), an abstract class that itself extends `ViewModelBase` and implements `IMainViewModel`. From there this class inherits the WPF main-window plumbing: it hooks `Application.Current.MainWindow.Closing`, supports cancelling the close via the overridable `OnClosing()`, and provides `Shutdown(exitCode)`, `CloseWindow()`, and the window `Icon`. `UserViewModel` overrides none of these; it inherits them as-is so that a host window bound to this view model gets standard close/shutdown handling.
- Implements `IUserViewModel`, whose two members (`AddUserViewModel`, `ShowUsersViewModel`) are satisfied by this class's two imported properties.
- `[Export(typeof(IUserViewModel))]` and `[Export(typeof(IMainViewModel))]`: the class registers itself in the MEF container under both contracts, so it can be resolved as the feature's root and as a generic main view model.
- Both properties are MEF `[Import]`s, filled at composition time from the container:
  - `IAddUserViewModel` → `AddUserViewModel` (`[Export(typeof(IAddUserViewModel))]` on `AddUserViewModel` in the same folder)
  - `IShowUsersViewModel` → `ShowUsersViewModel` (`[Export(typeof(IShowUsersViewModel))]` on `ShowUsersViewModel` in the same folder)
- `UserManagement/Design/DesignUserViewModel.cs` subclasses it for the XAML designer, constructing `DesignAddUserViewModel` / `DesignShowUsersViewModel` directly in place of the container imports.

### Behavior & Details

- The class is `partial` but the only part present here carries everything — the attributes, the base type, the interface, and the two properties. There is no other partial file in the project.
- Both imported properties are initialized to `null!` (the null-forgiving operator). That is just a compile-time appeasement: before composition, the backing fields really are `null`, and the class does not check for that. Anyone constructing a `UserViewModel` by hand (as the unit test does) must assign both properties before use; the MEF-composed instance is safe because the container populates them.
- It adds no commands, state, or messaging. Child behavior lives entirely in `AddUserViewModel` (validation, `SaveUserCommand`) and `ShowUsersViewModel` (loading the user list, reacting to `UserChangedMessage` via `WeakReferenceMessenger`). Anything bound in a view against `UserViewModel` is really going through these children.

### Rough edges

- `DesignUserViewModel` (the XAML designer substitute) assigns both child properties itself, which is fine; but note the base designer guard only exists in `MainViewModelBase`'s constructor for the main-window/icon setup — nothing here protects a design-time `UserViewModel` from the `MainViewModelBase` constructor touching `Application.Current` if design mode detection fails.