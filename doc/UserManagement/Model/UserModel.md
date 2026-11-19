# UserManagement/Model/UserModel.cs

### Purpose

`UserModel` is the view-side representation of a user in the UserManagement module: a small observable model holding just `FirstName` and `LastName`, shaped for data binding. It exists so the UI can display users and react to changes without depending on the business layer's entity types (`User` in `BusinessLayer`) or the `UserDto` used for service calls.

### Role in the Codebase

This class is the sole occupant of `UserManagement/Model/` and the only display object in the module. It contains no business logic, validation, or persistence — just the two name fields the form shows. The dependency flow is one-way: view models and views consume `UserModel`; nothing writes into it through a service or repository.

It is used in two distinct ways:

- As **display data** inside `ShowUsersViewModel.Users` (an `ObservableCollection<UserModel>`).
- As a **message payload**: `AddUserViewModel` constructs a fresh `UserModel` after a successful save and ships it out in a `UserChangedMessage`, which `ShowUsersViewModel` registers to receive and appends to the collection.

### Relationships

- **Base class:** `ObservableObject` from the external `CommunityToolkit.Mvvm` library. The `partial` keyword and the `[ObservableProperty]` attributes indicate the source generator derives the public properties and the `INotifyPropertyChanged` plumbing; this file provides only the backing fields.

- **Consumers:**
  - `UserManagement/ViewModel/ShowUsersViewModel.cs` — creates instances by mapping `User` objects from `UserService.GetActiveUsers()` onto `FirstName`/`LastName`, and adds message-carried instances on `UserChangeType.Added`.
  - `UserManagement/ViewModel/AddUserViewModel.cs` — instantiates one from the form fields after `UserService.AddUser` succeeds, and wraps it in `UserChangedMessage(UserChangeType.Added, ...)`.
  - `UserManagement/Messaging/UserChangedMessage.cs` — carries the model as `ValueChangedMessage<UserModel>`'s `Value`.
  - `UserManagement/Design/DesignShowUsersViewModel.cs` — creates three hard-coded instances (Jane Doe, John Smith, Alice Johnson) to give the designer demo data.
  - `WinFormsApp1/View/UserForm.cs` — binds the list box to `ShowUsersViewModel.Users` and formats each item as `"{FirstName} {LastName}"`.

### Behavior & Details

- Two private fields, `_firstName` and `_lastName`, each initialized to `string.Empty`. The `[ObservableProperty]` source generator turns each into a public auto-property named `FirstName` / `LastName` (first letter capitalized, underscore dropped) that raises `PropertyChanged` on assignment. That notification is what lets the UI refresh when a value changes.
- There is no `Id` property. `UserModel` instances are value-only display objects, so code cannot match a model back to a persisted user.
- Instances are typically created via object initializers (`new UserModel { FirstName = ..., LastName = ... }`); no custom constructor is defined.

### Rough edges

Because `UserModel` carries no identifier, `ShowUsersViewModel`'s messenger handler has no way to locate the existing entry for a user — its `Updated` and `Deleted` cases are both `throw new NotImplementedException()`. Only `Added` is currently supported, and an update or deletion broadcast would crash the app rather than fail gracefully.