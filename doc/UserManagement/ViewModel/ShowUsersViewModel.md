# `ShowUsersViewModel` (`UserManagement/ViewModel/ShowUsersViewModel.cs`)

### Purpose

`ShowUsersViewModel` is the view model behind the user-list half of the UserManagement feature. It holds the collection of users the UI displays, fetches that list from the business layer on demand, and listens for `UserChangedMessage` broadcasts so the list stays in sync when a sibling view model adds a user.

### Role in the Codebase

This class is one of two child view models in `UserManagement/ViewModel/`, the other being `AddUserViewModel` for creating users. It is the implementation half of the `IShowUsersViewModel`/`ShowUsersViewModel` pair that every view model in this folder follows.

The class carries `[Export(typeof(IShowUsersViewModel))]`, which makes it visible to MEF. The parent `UserViewModel` (same folder) has an `[Import]` property of type `IShowUsersViewModel`, so the composition container wires this class into the parent without either side referencing the concrete type. Downstream, `WpfApp1/View/UserView.xaml` binds the `Users` collection to a list and triggers `LoadedCommand` on load.

### Relationships

- **`ViewModelBase`** (`Mvvm/ViewModelBase.cs`) — the base class. It is an abstract `ObservableObject` that adds design-mode detection and a child-view-model registry (`RegisterChildViewModel`/`OnClosing`/`OnClosed`). `ShowUsersViewModel` inherits the `ObservableObject` machinery from it (through `ViewModelBase`) and adds nothing to the child-registry features.
- **`IShowUsersViewModel`** (`UserManagement/ViewModel/IShowUsersViewModel.cs`) — the contract implemented here. It declares `Users`, the two `[Import]` services, the generated `LoadedCommand`, and `OnLoadedAsync()`. Note the interface also exposes `UserService` and `LoggingService`, so MEF must import them through this class even though `LoggingService` is never called in it.
- **`IUserService` / `ILoggingService`** — imported via `[Import]` properties, satisfied by MEF from `BusinessLayer/UserService.cs` and the Logging module respectively.
- **`UserChangedMessage` / `UserChangeType`** (`UserManagement/Messaging/`) — the message this class subscribes to. The producer is `AddUserViewModel`, which sends `new UserChangedMessage(UserChangeType.Added, userModel)` on `WeakReferenceMessenger.Default` after a successful save.
- **`UserModel`** (`UserManagement/Model/UserModel.cs`) — a simple `ObservableObject` with `FirstName`/`LastName`; the element type of the `Users` collection.
- **`UserDto`** (`Dto`) — the return type of `IUserService.GetActiveUsers()`, mapped into `UserModel` here.

### Behavior & Details

**Construction.** The constructor registers `this` as a recipient of `UserChangedMessage` on `WeakReferenceMessenger.Default`. Because the registration is weakly referenced, the view model stays alive only while something else (the `UserViewModel` parent) holds it — a deliberate pattern in this codebase to avoid messenger leaks. The comment in the constructor says "UserLoggedInMessage" but the registered type is `UserChangedMessage`; the comment is stale.

**`Users`.** A read-only `ObservableCollection<UserModel>` initialized empty at declaration. The UI binds to it directly; it is populated by both `OnLoadedAsync` and the message handler. There is no deduplication — if the same user arrives through both paths, it appears twice.

**`OnLoadedAsync`.** Backed by `[RelayCommand]`, so the source generator creates `LoadedCommand` (the `IAsyncRelayCommand` on the interface). It awaits `UserService.GetActiveUsers()`, maps each `UserDto` to a fresh `UserModel` (only `FirstName` and `LastName` are copied; `UserDto.Id` is dropped), and appends them to `Users`. It appends rather than clears, so calling it twice doubles the list — the intended trigger is the WPF view's one-time Loaded event.

**Message handling.** The registered handler switches on `message.UserChangeType`:

- `Added` — appends `message.Value` to `Users`. This is the only implemented case; `AddUserViewModel` is the sole current sender and always uses `Added`.
- `Updated` and `Deleted` — both throw `NotImplementedException`. No code in this repository currently sends those message types, but if a future sender does, the exception propagates through the messenger and crashes the message dispatch.

**`UserService` / `LoggingService`.** Both are `[Import]`-required (`null!`-backed). `UserService` is used by `OnLoadedAsync`. `LoggingService` is never invoked anywhere in this class — the import exists to satisfy the interface contract.

### Rough edges

- The `Updated`/`Deleted` branches of the message handler throw `NotImplementedException`, so any future message of those types is an unhandled crash rather than a no-op.
- `OnLoadedAsync` appends to `Users` without clearing it first, so repeated invocations duplicate entries.
- The constructor comment references a `UserLoggedInMessage` that does not match the `UserChangedMessage` it actually registers for.
- `LoggingService` is imported but unused, and the mapping in `OnLoadedAsync` silently drops `UserDto.Id`, which the other consumers of the business layer keep.