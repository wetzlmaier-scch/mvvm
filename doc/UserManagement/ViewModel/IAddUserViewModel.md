I have what I need. Here is the documentation.

---

**File:** `UserManagement/ViewModel/IAddUserViewModel.cs`

### Purpose

`IAddUserViewModel` is the contract for the "add a new user" screen in the User Management module. It tells the rest of the application (and the WPF designer) what the add-user view model exposes: the editable user fields, the command that saves, and the services it needs to save correctly. It exists so callers can depend on the view model by interface and so the concrete view model is unit-testable and designer-friendly.

### Role in the Codebase

This is the interface half of the add-user view model pair. The concrete implementation is `AddUserViewModel` in the same `ViewModel` directory, which is exported via MEF (`[Export(typeof(IAddUserViewModel))]`) and discovered through this interface. The sibling interfaces `IUserViewModel`, `IShowUsersViewModel` follow the same "interface + implementation" convention; `IAddUserViewModel` is the one that owns the create-user form. Remove it and `AddUserViewModel` has nothing to implement, and consumers that take the interface can no longer compile.

### Relationships

- **Extends two parents:**
  - `IUser` (defined in `Dto/IUser.cs`) — contributes `FirstName` and `LastName` as `string` properties. This makes the view model itself carry the user data being entered.
  - `INotifyDataErrorInfo` (from `System.ComponentModel`) — the WPF standard validation surface that lets the UI bind per-property validation errors.

- **Members and their implementation sites** (in `AddUserViewModel`):
  - `UserService` — an `IUserService` (defined in `BusinessLayer/IUserService.cs`) injected via MEF. Used to persist the user.
  - `LoggingService` — an `ILoggingService` injected via MEF.
  - `SaveUserCommand` — an `IAsyncRelayCommand` (from `CommunityToolkit.Mvvm.Input`). The implementation generates this automatically from `SaveUserAsync()` by stripping the `Async` suffix.
  - `SaveUserAsync()` — the actual save operation.

- **Consumers** (depend on this interface, not the class):
  - `IUserViewModel` / `UserViewModel` — expose `AddUserViewModel` as an `IAddUserViewModel` so the shell view model can hand it to the view.
  - `DesignAddUserViewModel` (in `UserManagement/Design`) inherits from the concrete `AddUserViewModel` to feed sample data to the WPF designer; it does not implement the interface directly.

### Behavior & Details

The interface itself is pure contract — no logic. The behavior worth knowing lives in the one implementation, `AddUserViewModel`:

- It derives from `ValidatableViewModelBase<IUser>` (in `Mvvm/ValidatableViewModelBase.cs`), which brings the `INotifyDataErrorInfo` support and a `UserValidator`. The `FirstName`/`LastName` fields are `ObservableProperty`s and re-validate on change.
- `SaveUserAsync()` runs in this order:
  1. Validates all properties; if `HasErrors` is true it shows an error message and returns without saving.
  2. Calls `UserService.AddUser(new UserDto { ... })` to persist.
  3. Broadcasts a `UserChangedMessage` of type `UserChangeType.Added` via `WeakReferenceMessenger` so other parts of the app (e.g. the user list) can refresh.
  4. Clears `FirstName`/`LastName` back to empty, ready for the next entry.
- It catches `BusinessLayerException` and re-throws it, so business failures propagate to the caller rather than being swallowed.

Because `IAddUserViewModel` inherits `IUser`, any consumer can read the in-progress `FirstName`/`LastName` directly off the interface — the form data is the view model.

### Rough edges

The interface declares a settable `UserService` and `LoggingService`, but the only production implementation sets both via MEF `[Import]` and never assigns them through the setters. The settable accessors exist to satisfy MEF's injection, not to be set by callers — a reader should not treat them as a public API for swapping services at runtime.