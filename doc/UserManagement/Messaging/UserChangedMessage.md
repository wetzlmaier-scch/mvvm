### Purpose

`UserChangedMessage` (`UserManagement/Messaging/UserChangedMessage.cs`) is a message class that announces a change to a user — added, updated, or deleted — to other parts of the UserManagement module. It exists so that screens displaying user data can stay in sync with the user service without directly referencing it: a view model performs the change and sends a message; any view model that cares receives it through the CommunityToolkit `WeakReferenceMessenger`.

### Role in the Codebase

This class is the messaging contract of the `UserManagement/Messaging` folder, alongside its sibling `UserChangeType` enum (`UserManagement/Messaging/UserChangeType.cs`), which defines the three operations: `Added`, `Updated`, `Deleted`. It is a data model for events, not logic — its entire job is to carry a `UserModel` plus the kind of change that happened to it. Removing it would break both the sender (`AddUserViewModel`) and the receiver (`ShowUsersViewModel`), which both reference it directly.

### Relationships

- **Base type:** `ValueChangedMessage<UserModel>` from the external CommunityToolkit.Mvvm library. It supplies the `Value` property holding the `UserModel`, so `UserChangedMessage` only adds the change-type dimension.
- **`UserChangeType`** (same folder) — the enum naming the operation.
- **`UserModel`** (`UserManagement/Model/UserModel.cs`) — the payload, an `ObservableObject`-based model of a user.
- **Sender:** `UserManagement/ViewModel/AddUserViewModel.cs` builds the message after creating a user:
  ```csharp
  WeakReferenceMessenger.Default.Send(new UserChangedMessage(UserChangeType.Added, userModel));
  ```
- **Receiver:** `UserManagement/ViewModel/ShowUsersViewModel.cs` registers in its constructor and reacts by appending `message.Value` to its `Users` collection when the type is `Added`.

### Behavior & Details

- The primary primary-constructor takes the change type and the user. The convenience constructor that takes only a `UserModel` defaults to `UserChangeType.Added`. Any call site relying on "no type" therefore silently publishes an `Added` event — use the explicit constructor for anything else.
- `UserChangeType` is a read-only property (`{ get; }`) initialized once via the primary constructor; the message is immutable.
- `message.Value` (inherited from `ValueChangedMessage<UserModel>`) carries the user payload.

### Rough edges

`ShowUsersViewModel`'s handler throws `NotImplementedException` for `Updated` and `Deleted`, so only the `Added` path is actually handled today despite the enum advertising all three. Also, the single-argument constructor defaulting to `Added` can mask intent if a future sender forgets to pass the type.