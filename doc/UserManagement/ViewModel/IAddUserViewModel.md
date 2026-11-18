### Purpose

`UserManagement/ViewModel/IAddUserViewModel.cs`

`IAddUserViewModel` is the contract for the "add a new user" screen in the User Management module. It defines what the rest of the application can rely on from that screen: the user fields being entered, the command that saves them, and the service dependencies the screen needs. It exists so that consumers (and the WPF designer) depend on a stable abstraction rather than on the concrete view model.

### Role in the Codebase

This is the interface half of the add-user view model pair in `UserManagement/ViewModel`. The concrete implementation is `AddUserViewModel` in the same folder, which carries `[Export(typeof(IAddUserViewModel))]` and is therefore injected via MEF wherever the interface is imported. Removing this interface would break `AddUserViewModel`'s export/inheritance, the `AddUserViewModel` property on `IUserViewModel`, and `UserViewModel`'s corresponding `[Import]`.

### Relationships

- **`INotifyDataErrorInfo`** (`System.ComponentModel`) — inherited, not defined here. This is what makes the form's per-field validation visible to XAML binding (`Errors` collection), and it is the mechanism `AddUserViewModel`'s `ValidatableViewModelBase` implementation plugs into.
- **`IUser`** (defined in `Dto/IUser.cs`) — inherited. It declares only `FirstName` and `LastName`, so the form's data fields are part of this contract: any consumer can read the in-progress user values straight off the interface without knowing the concrete type.
- **`IUserService`** (`BusinessLayer/IUserService.cs`) — exposed as a settable property; the implementation imports it via `[Import]` and uses `AddUser(UserDto)` to persist a user.
- **`ILoggingService`** (`Logging/ILoggingService.cs`) — same pattern; imported and set by MEF in the implementation.
- **`IAsyncRelayCommand SaveUserCommand`** — the generated async command from `AddUserViewModel`'s `[RelayCommand]`-annotated `SaveUserAsync()`. Views bind to `SaveUserCommand` rather than calling `SaveUserAsync` directly.
- **Consumers:** `IUserViewModel` / `UserViewModel` expose the implementation as `IAddUserViewModel AddUserViewModel` so the shell view model can hand the screen to its view.

### Behavior & Details

The interface has no logic of its own; its shape constrains the implementation in a few ways worth knowing:

- `UserService` and `LoggingService` are `{ get; set; }`, while `SaveUserCommand` is get-only. The writable service properties are an accommodation for MEF: `AddUserViewModel` declares them as `[Import]` properties, which requires setters at the class level, so the interface mirrors that to keep the import surface part of the contract.
- `SaveUserAsync()` is part of the contract in addition to the command, so a consumer can trigger the save programmatically (e.g., from a parent view model) without going through the command.
- Because the interface extends `IUser`, the "form data is the view model" pattern holds here: `AddUserViewModel` implements `ValidatableViewModelBase<IUser>` and derives from the same `IUser`, so the `FirstName`/`LastName` properties it generates satisfy this interface directly.
- Note that the implementation also imports `IMessageService` and sends a `UserChangedMessage` after a successful save, but neither of those is on the interface — a consumer of `IAddUserViewModel` cannot observe that notification through the contract; it is only visible by going through the shared `WeakReferenceMessenger` channel.