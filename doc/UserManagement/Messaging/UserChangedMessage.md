# `UserChangedMessage` (`UserManagement/Messaging/UserChangedMessage.cs`)

### Purpose
`UserChangedMessage` is the message type that announces a change to a user — added, updated, or deleted — to other parts of the UserManagement module. It exists so that sibling view models can stay in sync without referencing each other: `AddUserViewModel` broadcasts one of these messages after saving a user, and `ShowUsersViewModel` listens for them to keep its displayed user list current.

### Role in the Codebase
This class is the single event payload in the `UserManagement/Messaging` folder; its only companion there is the `UserChangeType` enum, which names the kinds of change a message can carry. Together they are the "event contract" layer of the module: everything about what a user change looks like is defined here, while the sending and receiving logic lives in the view models. Nothing in the module's messaging can function without this class, since it is both the type that `WeakReferenceMessenger` dispatches and the type receivers register against.

### Relationships
- **Base type:** `ValueChangedMessage<UserModel>` from the external CommunityToolkit.Mvvm library. The base class supplies the `Value` property holding the `UserModel`, so `UserChangedMessage` only adds the change-type discriminator on top.
- **`UserChangeType`** (`UserManagement/Messaging/UserChangeType.cs`) — an enum with `Added`, `Updated`, and `Deleted`. It is the constructor parameter and the value of the `UserChangeType` property.
- **`UserModel`** (`UserManagement/Model/UserModel.cs`) — the user instance the message carries; it is passed straight through to the base class.
- **Sender:** `AddUserViewModel` (`UserManagement/ViewModel/AddUserViewModel.cs`) constructs `new UserChangedMessage(UserChangeType.Added, userModel)` in `SaveUserAsync()` and sends it via `WeakReferenceMessenger.Default.Send(...)` after `UserService.AddUser` succeeds.
- **Receiver:** `ShowUsersViewModel` (`UserManagement/ViewModel/ShowUsersViewModel.cs`) registers for `UserChangedMessage` in its constructor and switches on `message.UserChangeType`, appending `message.Value` to its `Users` collection for `Added`.

### Behavior & Details
The class is a primary constructor plus one property; it holds no state beyond what the base class and the constructor parameters provide.

- **Primary constructor** `UserChangedMessage(UserChangeType changeType, UserModel value)` forwards `value` to `ValueChangedMessage<UserModel>` and stores `changeType` in the read-only `UserChangeType` property.
- **Convenience constructor** `UserChangedMessage(UserModel value)` delegates to the primary one with `UserChangeType.Added`. No caller in this repository currently uses the overload; it encodes "a bare `UserModel` means a new user" as the default.
- **`UserChangeType` property** is get-only and set once at construction, so the kind of change is fixed for the message's lifetime.

Because `WeakReferenceMessenger` references recipients weakly, the receiver view model must stay alive externally (via MEF composition) for messages to arrive; the message itself has no lifetime or disposal behavior.

Note that the enum's `Updated` and `Deleted` cases are part of the contract but currently unhandled on the receiving side — `ShowUsersViewModel`'s handler throws `NotImplementedException` for both, so sending a message of those types today would fail in the handler rather than update the UI.