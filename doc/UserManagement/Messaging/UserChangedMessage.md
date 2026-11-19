# `UserChangedMessage` (`UserManagement/Messaging/UserChangedMessage.cs`)

### Purpose

`UserChangedMessage` is the message type that announces a change to a user — added, updated, or deleted — to other parts of the UserManagement module. It exists so that sibling view models can stay in sync without knowing about each other: `AddUserViewModel` doesn't reference `ShowUsersViewModel`, it just broadcasts "a user changed" and whoever is listening reacts.

### Role in the Codebase

This class is one of only two members of `UserManagement/Messaging/`, the module's event-contract layer; the other is `UserChangeType`, its companion enum. Together they form the loose-coupling point between the module's view models, which otherwise communicate through the business layer. If it were removed, `AddUserViewModel` would have no way to tell `ShowUsersViewModel` about new users, and the user list in the UI would not update after a save.

### Relationships

- **Base type:** `ValueChangedMessage<UserModel>` from the external CommunityToolkit.Mvvm library. The base class supplies the `Value` property holding the `UserModel`, so `UserChangedMessage` only adds the change-type information.
- **`UserChangeType`** (`UserManagement/Messaging/UserChangeType.cs`) — the enum (`Added`, `Updated`, `Deleted`) that classifies the message. Required by the primary constructor and exposed as a property.
- **`UserModel`** (`UserManagement/Model/UserModel.cs`) — the observable data model (`FirstName`, `LastName`) carried in `Value`.
- **Sender:** `AddUserViewModel` (`UserManagement/ViewModel/AddUserViewModel.cs`) constructs `new UserChangedMessage(UserChangeType.Added, userModel)` in `SaveUserAsync()` and sends it via `WeakReferenceMessenger.Default` after a successful `UserService.AddUser` call.
- **Receiver:** `ShowUsersViewModel` (`UserManagement/ViewModel/ShowUsersViewModel.cs`) registers for `UserChangedMessage` in its constructor and switches on `message.UserChangeType`, appending `message.Value` to its `Users` collection for `Added`.

### Behavior & Details

The class has no logic of its own — it is a simple data carrier with two constructors and one read-only property:

- **Primary constructor** `UserChangedMessage(UserChangeType changeType, UserModel value)` — a primary constructor that forwards `value` to the base `ValueChangedMessage<UserModel>` and stores `changeType` in the read-only `UserChangeType` property.
- **Convenience constructor** `UserChangedMessage(UserModel value)` — delegates to the primary one with `UserChangeType.Added`. No caller in this repository currently uses the overload; it encodes "a bare user is an addition" as a default.

Two caveats worth knowing:

- Only `UserChangeType.Added` is fully implemented end-to-end. `ShowUsersViewModel`'s handler throws `NotImplementedException` for `Updated` and `Deleted`, so sending those today would crash the receiving view model, not degrade gracefully.
- Registration is via `WeakReferenceMessenger`, so the message payload here is not what keeps recipients alive — the view model is. The message itself is created, sent, and discarded in `SaveUserAsync()`.

### Rough edges

- The single-argument constructor's default of `Added` is untested and currently unused; it is easy for a future caller to send a message that silently claims "added" when they meant something else, rather than naming the change explicitly.
- The class name and the property name are both `UserChangeType` (the property shadows the enum type within the class), which is legal but mildly confusing when reading `message.UserChangeType`.