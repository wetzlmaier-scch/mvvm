# UserManagement/Model/UserModel.cs

### Purpose

`UserModel` is the view-side representation of a user: a tiny MVVM model holding just a `FirstName` and a `LastName`, shaped for data binding in the UserManagement module. It exists so the UI layer can display and pass around user display data without touching the persistence layer's types (like `UserDto` or the Hibernate entities). The `[ObservableProperty]` attributes from CommunityToolkit.Mvvm make each field into an auto-generated public property that raises `PropertyChanged`, so bound controls refresh when a value changes.

### Role in the Codebase

This is the only file in `UserManagement/Model/`, making it the module's sole data model. It is a display object, not an entity — the UserManagement project has no storage logic of its own, so this class carries no validation or persistence concerns. Removing it would break the view model, messaging, and WinForms view layers described below, since they all hold and render `UserModel` instances.

### Relationships

- **Base class:** `ObservableObject` from the external `CommunityToolkit.Mvvm.ComponentModel` package (the package is referenced in the `Mvvm` project, and this class is marked `partial` because the toolkit's source generator fills in the rest).
- **`UserManagement/ViewModel/ShowUsersViewModel.cs`** — keeps an `ObservableCollection<UserModel>` of displayed users. It creates the instances by mapping `User` objects from `UserService.GetActiveUsers()` into `new UserModel { FirstName = ..., LastName = ... }`, and appends to it when it receives a `UserChangedMessage` of type `Added`.
- **`UserManagement/ViewModel/AddUserViewModel.cs`** — after saving a new user via `UserService.AddUser`, it constructs a `UserModel` from the form fields and sends it out as `UserChangedMessage(UserChangeType.Added, userModel)` so `ShowUsersViewModel` picks it up.
- **`UserManagement/Messaging/UserChangedMessage.cs`** — wraps a `UserModel` (it extends `ValueChangedMessage<UserModel>`), so messages about user changes always carry a `UserModel`.
- **`UserManagement/Design/DesignShowUsersViewModel.cs`** — creates hardcoded `UserModel` instances to give the XAML designer demo data.
- **`WinFormsApp1/View/UserForm.cs`** — binds the user list box to `ShowUsersViewModel.Users` and formats each `UserModel` as `"{FirstName} {LastName}"` in its `lstUsers.Format` handler.

### Behavior & Details

Two private fields, `_firstName` and `_lastName`, both defaulting to `string.Empty`. The `[ObservableProperty]` source generator turns each into a public `string` property named `FirstName` / `LastName` and emits the property-change notifications — that generated code is what every consumer actually accesses, including object-initializer syntax like `new UserModel { FirstName = "Jane", LastName = "Doe" }`.

There is no other logic: no validation, no `Equals`/`GetHashCode` overrides, and the two fields do not reference each other. Instances are short-lived display objects — typically constructed fresh per user by `ShowUsersViewModel` or `AddUserViewModel` rather than shared long-term.

### Rough edges

The model is a flat copy of data with no identity (no `Id` property), so nothing in the UI can distinguish two users by anything but name. This is directly implicated by `ShowUsersViewModel`, where the `Updated` and `Deleted` cases of its message handler are `NotImplementedException` — without an identifier there is no natural way to locate the row to update or remove once those message types are actually sent.