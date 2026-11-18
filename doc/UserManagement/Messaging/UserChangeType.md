# `UserManagement/Messaging/UserChangeType.cs`

### Purpose

This file defines `UserChangeType`, the enum that classifies *what happened* to a user whenever one is changed. It exists so that the messaging layer can carry the kind of change (`Added`, `Updated`, `Deleted`) alongside the affected `UserModel` in a single message, letting recipients react differently to each kind.

### Role in the Codebase

It is a small shared vocabulary type for the `UserManagement/Messaging` namespace, whose other member is `UserChangedMessage`. The enum is the discriminating field of that message: `UserChangedMessage` wraps a `UserChangeType` plus a `UserModel`, and recipients branch on it. Remove it and `UserChangedMessage` cannot express which of the three operations occurred.

### Relationships

- **Defined alongside:** `UserChangedMessage` (`UserManagement/Messaging/UserChangedMessage.cs`) — takes a `UserChangeType` in its constructor, exposes it as its `UserChangeType` property, and its convenience constructor defaults a new user to `UserChangeType.Added`.
- **Producer:** `AddUserViewModel` (`UserManagement/ViewModel/AddUserViewModel.cs`) sends `new UserChangedMessage(UserChangeType.Added, userModel)` through `WeakReferenceMessenger` after a successful save.
- **Consumer:** `ShowUsersViewModel` (`UserManagement/ViewModel/ShowUsersViewModel.cs`) registers a handler for `UserChangedMessage` and switches on `message.UserChangeType`, adding to its `Users` collection on `Added`.

### Behavior & Details

Three members, all implicitly valued:

- `Added` — a user was created.
- `Updated` — an existing user changed.
- `Deleted` — a user was removed.

Because it is a plain enum, `exhaustive` handling is up to the receiver. Note that `ShowUsersViewModel`'s handler currently only implements `Added`; the `Updated` and `Deleted` cases throw `NotImplementedException`, so a recipient registered for all message kinds will crash on those two if they are ever sent. As of the code in this repository, only `Added` is actually sent, so the other two members are reserved for use that has not been implemented yet.

### Rough edges

`Updated` and `Deleted` exist as values, but the only registered consumer (`ShowUsersViewModel`) throws `NotImplementedException` for both, and no code currently sends either — so these members describe operations the messaging layer cannot yet handle end to end.