### Purpose

`UserManagement/Messaging/UserChangeType.cs` defines `UserChangeType`, an enum that names *what happened* to a user: `Added`, `Updated`, or `Deleted`. It exists so that a message can carry not just "a user changed" but the specific kind of change, letting the recipient take a different action per case without the sender knowing who is listening.

### Role in the Codebase

It is one of the two types in the `UserManagement/Messaging` folder, the small event contract of the UserManagement module. The companion file `UserChangedMessage.cs` is the actual payload that travels through the message bus and cannot be constructed without a `UserChangeType`. Removing this enum would force `UserChangedMessage` and both its sender and receiver to be rewritten; there is no other mechanism in the module that distinguishes add from update from delete.

### Relationships

- **`UserChangedMessage`** (`UserManagement/Messaging/UserChangedMessage.cs`) — the direct dependent. Its primary constructor takes a `UserChangeType` and stores it in a read-only `UserChangeType` property; its single-argument convenience constructor hardcodes `UserChangeType.Added`.
- **Sender:** `AddUserViewModel` (`UserManagement/ViewModel/AddUserViewModel.cs`) sends `new UserChangedMessage(UserChangeType.Added, userModel)` over `WeakReferenceMessenger.Default` after successfully saving a new user.
- **Receiver:** `ShowUsersViewModel` (`UserManagement/ViewModel/ShowUsersViewModel.cs`) registers for `UserChangedMessage` in its constructor and switches on `message.UserChangeType`.

It has no dependencies of its own — just a namespace declaration and the enum.

### Behavior & Details

- Three members, with the implicit default integral values: `Added = 0`, `Updated = 1`, `Deleted = 2`. The values are not serialized or persisted anywhere in the repository, so only the names matter.
- The enum is `public` and lives in the module's `Messaging` namespace, so any code in the app that participates in the user-change flow can use it.
- Note the current reality on the consumer side: `ShowUsersViewModel` handles only `Added` (it appends to the displayed list). Both `Updated` and `Deleted` cases currently end in `throw new NotImplementedException()`. So today, sending a message with either of those values will crash the first handler that receives it — the enum advertises a fuller contract than the codebase has implemented. The convenience constructor on `UserChangedMessage` also silently defaults to `Added`, so a sender that forgets to specify a change type produces "Added" rather than an error.

### Rough edges

- `Updated` and `Deleted` are declared but unimplemented: `ShowUsersViewModel` throws `NotImplementedException` for both, so the enum currently over-promises relative to what any handler can actually process.