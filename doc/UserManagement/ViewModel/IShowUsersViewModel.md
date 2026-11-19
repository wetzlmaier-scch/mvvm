### Purpose

`UserManagement/ViewModel/IShowUsersViewModel.cs` defines `IShowUsersViewModel`, the contract for the view model that displays the list of active users in the UserManagement module. The interface exists so the parent view model (`UserViewModel`) can hold and expose the show-users feature through an abstraction rather than a concrete class, which is what makes MEF-based composition possible in this folder.

### Role in the Codebase

This is the interface half of the interface/implementation pair pattern used throughout `UserManagement/ViewModel/`: each feature has an `I*ViewModel.cs` contract alongside its concrete `*ViewModel.cs`. Here the implementation is `ShowUsersViewModel` in the same folder, which carries `[Export(typeof(IShowUsersViewModel))]` so the MEF container wires it in under this contract. `IUserViewModel` (same folder) also exposes the feature as a read-only `IShowUsersViewModel ShowUsersViewModel` property. If this interface were removed, the parent view model and the MEF export/import pair would have nothing to compose against.

### Relationships

- **`UserManagement/ViewModel/ShowUsersViewModel.cs`** — the sole implementation. It is `partial`, extends `ViewModelBase` (an `Mvvm`-namespace base type), and satisfies every member declared here. Its `UserService` and `LoggingService` properties carry `[Import]`, so MEF supplies them after construction.
- **`UserManagement/ViewModel/UserViewModel.cs`** — the parent view model; it declares `[Import] public IShowUsersViewModel ShowUsersViewModel { get; set; }`, so MEF injects the implementation into it. `IUserViewModel` mirrors this property read-only.
- **`BusinessLayer/IUserService`** — the `UserService` property's type; defined in `BusinessLayer/IUserService.cs`. The implementation's `OnLoadedAsync()` calls its `GetActiveUsers()` member.
- **`Logging/ILoggingService`** — the `LoggingService` property's type; defined in `Logging/ILoggingService.cs` (extends `ILogger`). Notably, no call to `LoggingService` appears in `ShowUsersViewModel`, so the injected property is currently unused there.
- **`UserManagement.Model.UserModel`** — the element type of the `Users` collection; a small `ObservableObject` with `FirstName`/`LastName`.
- **`CommunityToolkit.Mvvm.Input.IAsyncRelayCommand`** — the type of `LoadedCommand`, produced by the CommunityToolkit source generator in the implementation.

A caller that depends on this contract is `UserViewModel` (via MEF `[Import]`); the XAML view for the user screen binds against the concrete class, not the interface.

### Behavior & Details

The members, and what each pins down:

- **`Users`** — a read-only `ObservableCollection<UserModel>`. The backing collection is created in the implementation and is never re-assigned; consumers bind to it, and the implementation mutates its contents (initial load plus incoming `UserChangedMessage`s).
- **`UserService`** / **`LoggingService`** — writable service properties. They are not injected through the constructor; MEF sets them via `[Import]` after construction, so any code path that touches them before composition completes would hit the `null!` defaults.
- **`LoadedCommand`** — an `IAsyncRelayCommand` that the view invokes when the users page loads. The implementation generates it from `OnLoadedAsync()` with the `[RelayCommand]` attribute, so the method name and the command property must stay in sync.
- **`OnLoadedAsync()`** — the load operation behind `LoadedCommand`: awaits `UserService.GetActiveUsers()`, maps each `UserDto` to a new `UserModel` (copying `FirstName` and `LastName` only), and appends the results to `Users`. It appends rather than resets, so a second invocation would duplicate the list.

Because `LoadedCommand` is the generated wrapper for `OnLoadedAsync`, renaming or changing that method's signature changes the command's shape; conversely, the interface only names the command, so the implementation could in principle construct it by hand — it doesn't.

### Rough edges

The `UserChangeType.Updated` and `UserChangeType.Deleted` message cases in the implementation throw `NotImplementedException`; the interface's `Users` collection therefore only ever grows in the current state of the code, and a live update or deletion of a user will crash the handler.