# `UserManagement/ViewModel/ShowUsersViewModel.cs`

### Purpose

`ShowUsersViewModel` is the ViewModel for the "show users" part of the user-management feature. It holds the list of users the UI displays (`Users`), fetches that list from the business layer on load, and keeps the list updated when other parts of the app add users via the `UserChangedMessage` messenger.

### Role in the Codebase

It lives in `UserManagement/ViewModel/` alongside the other feature ViewModels (`UserViewModel`, `AddUserViewModel`) and is the concrete implementation of `IShowUsersViewModel` (defined in `UserManagement/ViewModel/IShowUsersViewModel.cs`). It is exported from the MEF composition container via `[Export(typeof(IShowUsersViewModel))]`, which is how `UserViewModel`'s `[Import] public IShowUsersViewModel ShowUsersViewModel` property gets populated. `UserViewModel` in turn exports itself as the main view model (`IUserViewModel` / `IMainViewModel`), so this class is one of the child view models the shell views bind to:

- `WpfApp1/View/UserView.xaml` binds a `ListView` to `ShowUsersViewModel.Users` and invokes `LoadedCommand` in a `DataTrigger`.
- `WinFormsApp1/View/UserForm.cs` binds a `ListBox` through a `BindingSource` to `Users` and executes `LoadedCommand` on the form's `Load` event.
- `ConsoleApp1/Program.cs` calls `ShowUsersViewModel.OnLoadedAsync()` directly, then prints `Users`.
- `TestProject1/UserViewModelTest.cs` instantiates it and tests both `OnLoadedAsync` and the message handler.
- `UserManagement/Design/DesignShowUsersViewModel.cs` subclasses it to seed demo data for the XAML designer.

Removing it would break the user list in both WPF and WinForms shells, the console app, and the test project.

### Relationships

- **Base class:** `ViewModelBase` (in `Mvvm/ViewModelBase.cs`), itself extending CommunityToolkit Mvvm's `ObservableObject` and implementing `IViewModel`. This class inherits the observable-property machinery and the child-view-model lifecycle helpers; it overrides none of them.
- **`[Import]` dependencies** (injected by MEF at composition time):
  - `IUserService` (`BusinessLayer/IUserService.cs`) — used for `GetActiveUsers()`, which returns `Task<List<UserDto>>`.
  - `ILoggingService` (`Logging`) — imported but currently never used in this class.
- **`UserModel`** (`UserManagement/Model/UserModel.cs`) — the observable view-model item with `FirstName`/`LastName` properties, stored in `Users`.
- **`UserChangedMessage`** (`UserManagement/Messaging/UserChangedMessage.cs`) — a `ValueChangedMessage<UserModel>` carrying a `UserChangeType`. Sent by `AddUserViewModel` (line 66 of that file) after a successful add.
- **`UserDto`** (`Dto`) — the business-layer user returned by `IUserService`; only `FirstName` and `LastName` are copied into `UserModel`.

### Behavior & Details

- **`Users`** — a read-only `ObservableCollection<UserModel>`, initialized inline. The only mutation paths are `OnLoadedAsync` (initial population) and the messenger handler (appending new users).
- **Constructor** — registers `this` as a recipient of `UserChangedMessage` on `WeakReferenceMessenger.Default` (registration is weakly referenced, so the VM must stay alive externally for messages to arrive — composition keeps it alive). The switch on `message.UserChangeType` currently handles only `Added` (appends `message.Value` to `Users`); `Updated` and `Deleted` throw `NotImplementedException`, so sending either would crash the recipient.
- **`OnLoadedAsync`** — marked `[RelayCommand]`, so the source generator produces `LoadedCommand` (an `IAsyncRelayCommand`) that the WPF view triggers on load and `UserForm` executes on the form's `Load`. It calls `await UserService.GetActiveUsers()` and adds one `UserModel` per returned `UserDto`, mapping only first and last name. It appends to the existing collection rather than clearing it first, so running the load twice duplicates entries. Errors from the service propagate to the caller of the command (e.g., the console app or the test).

### Rough edges

- The `LoggingService` import is declared but never used.
- `OnLoadedAsync` appends without clearing `Users`, so re-running the load command duplicates the whole list; only the initial load is expected by the current callers.
- The `Updated` and `Deleted` branches of the message handler throw `NotImplementedException`; nothing else in the codebase sends those types today, but it means the "list of users" cannot stay in sync with updates or deletions yet.