# `UserManagement/Messaging/UserChangeType.cs`

### Purpose

This file defines `UserChangeType`, an enum that classifies *what happened* to a user when one is changed: `Added`, `Updated`, or `Deleted`. It exists so the messaging layer can say not just that a user changed, but what kind of change it was, letting recipients react differently per case.

It is a small shared-vocabulary type — a few lines long with no logic — and its entire job is to be read as the discriminating field of the message that carries it.

### Role in the Codebase

The enum is one of only two members of the `UserManagement/Messaging` folder, the app's event-contract layer. The other is `UserChangedMessage`, and the relationship is one-way: the message is defined in terms of this enum and cannot function without it. If the enum were removed, `UserChangedMessage` would stop compiling, and the message-based refresh flow between the add-user screen and the user list would fall apart with it.

### Relationships

- **`UserChangedMessage`** (`UserManagement/Messaging/UserChangedMessage.cs`) — the direct dependent. It takes a `UserChangeType` in its primary constructor and exposes it as a read-only `UserChangeType` property. Its convenience constructor (user model only) hard-codes `UserChangeType.Added`.
- **Producer:** `AddUserViewModel` (`UserManagement/ViewModel/AddUserViewModel.cs`) sends `new UserChangedMessage(UserChangeType.Added, userModel)` through `WeakReferenceMessenger` after a successful user creation. This is the only place in the repository that constructs a message with an explicit change type.
- **Consumer:** `ShowUsersViewModel` (`UserManagement/ViewModel/ShowUsersViewModel.cs`) registers a `WeakReferenceMessenger` handler for `UserChangedMessage` and switches on `message.UserChangeType` to decide what to do with the received user.

### Behavior & Details

- Three values, in declaration order (and so with the underlying integer values 0, 1, 2): `Added`, `Updated`, `Deleted`. There is no memberwise logic to document beyond that.
- The `Updated` and `Deleted` members are currently defined but unused end-to-end. `ShowUsersViewModel`'s switch handles only `Added` (appending to its `Users` collection) and explicitly throws `NotImplementedException` for the other two. So anything that ever sends an `Updated` or `Deleted` message today would crash the recipient rather than be ignored.
- The message's no-type convenience constructor defaults to `Added`, so any future sender that omits the change type implicitly produces `UserChangeType.Added` rather than an error.

### Rough edges

The enum advertises three operations, but only `Added` is implemented anywhere. `Updated` and `Deleted` are dead values that a consuming switch treats as fatal (`NotImplementedException`), so adding a producer for them without also finishing the consumers would introduce a crash path, not a feature.