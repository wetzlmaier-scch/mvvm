# UserManagement/Model/UserModel.cs

### Purpose

`UserModel` is the view-side representation of a user in the UserManagement module: a tiny MVVM model holding just `FirstName` and `LastName`, shaped for WPF data binding. It exists so the UI layer can display and announce users with change notifications, decoupled from the business-layer `User` entity and the `UserDto` transfer type.

### Role in the Codebase

This is the only file in `UserManagement/Model/`, making it the module's sole data model. It is a display object, not an entity — the UserManagement project has no storage logic of its own, so this class carries nothing beyond what the UI shows. If it were removed, every view, view model, and message in the module would lose the type it binds to.

### Relationships

- **Base type:** `ObservableObject` from the external `CommunityToolkit.Mvvm` library. It supplies `INotifyPropertyChanged`; nothing is overridden here.
- **`UserManagement/ViewModel/ShowUsersViewModel.cs`** — keeps an `ObservableCollection<UserModel>` of displayed users. It creates the instances by mapping `User` objects from `UserService.GetActiveUsers()` in `OnLoadedAsync`, and appends to the collection when a `UserChangedMessage` of type `Added` arrives.
- **`UserManagement/ViewModel/AddUserViewModel.cs`** — after saving a new user via `UserService.AddUser`, it constructs a `UserModel` from the form fields and sends it out as `UserChangedMessage(UserChangeType.Added, userModel)`.
- **`UserManagement/Messaging/UserChangedMessage.cs`** — wraps a `UserModel` (it extends `ValueChangedMessage<UserModel>`), so messages about user changes always carry a `UserModel`.
- **`UserManagement/Design/DesignShowUsersViewModel.cs`** — creates hardcoded `UserModel` instances to give the XAML designer demo data.
- **`WinFormsApp1/View/UserForm.cs`** — binds the user list box to `ShowUsersViewModel.Users` and formats each `UserModel` as `"{FirstName} {LastName}"` in its `lstUsers.Format` handler.

### Behavior & Details

Two private fields, `_firstName` and `_lastName`, both defaulting to `string.Empty`. The `[ObservableProperty]` source generator turns each into a public `string` property named `FirstName` / `LastName` that raises `PropertyChanged` when assigned, which is what makes WPF bindings to these properties update automatically. There is no hand-written property code in this class — the `partial` keyword exists solely for the generated half.

The class has no `Id` or any identifier. That matters because `ShowUsersViewModel`'s messenger handler currently throws `NotImplementedException` for `Updated` and `Deleted` change types; without an identity on the model, matching an incoming message to the right item in `Users` would have no key to match on.