# `UserManagement/ViewModel/AddUserViewModel.cs`

### Purpose

`AddUserViewModel` is the view model behind the "add user" form. It holds the form fields (`FirstName`, `LastName`), validates them with FluentValidation, saves a new user through the business layer, and broadcasts a "user added" message to the rest of the application. It exists so that every UI front-end (WPF, WinForms, console) and the test suite can drive the same add-user behaviour without touching persistence.

### Role in the Codebase

It is one of the leaf view models of the `UserManagement` module. The parent `UserViewModel` exposes it as its `AddUserViewModel` property (the `IUserViewModel` interface declares `IAddUserViewModel AddUserViewModel { get; }`), and the WPF and WinForms user forms bind directly to its `FirstName`, `LastName`, and `SaveUserCommand`. Because it is the single place that performs validation, the `UserService.AddUser` call, the `UserChangedMessage` broadcast, and the field reset, removing it would break the add-user feature in every front-end and the `UserViewModelTest` suite.

### Relationships

- **Base class** — `ValidatableViewModelBase<T>` (`Mvvm/ValidatableViewModelBase.cs`) supplies `INotifyDataErrorInfo`-driven validation backed by a FluentValidation `IValidator<T>`. `AddUserViewModel` instantiates it with `IUser` and a `new UserValidator()`, so it inherits `GetErrors`, `HasErrors`, `ErrorsChanged`, `ValidateAllProperties`, and `ValidateProperty`. Nothing here overrides any of them.
- **`IUser`** (`Dto/IUser.cs`) — the interface the class implements via the base class generic; the generated `FirstName`/`LastName` properties satisfy it.
- **`UserValidator`** (`Dto/Validation/UserValidator.cs`) — FluentValidation rules requiring `FirstName` and `LastName` to be non-empty; constructed by this class and passed to the base.
- **`IAddUserViewModel`** (`UserManagement/ViewModel/IAddUserViewModel.cs`) — the MEF export contract. The class is exported as this interface via `[Export(typeof(IAddUserViewModel))]`, and the same interface is what `UserViewModel` imports.
- **`IUserService`** (`BusinessLayer/IUserService.cs`) — MEF-injected via `[Import]`; its `AddUser(UserDto)` performs the actual persistence.
- **`ILoggingService`** (`Logging`) and **`IMessageService`** (`Mvvm`) — also MEF-injected. `MessageService` is used to surface the validation failure; `LoggingService` is injected but not referenced by any method body in this class.
- **`UserDto`** (`Dto`) — the data object built from the form fields and passed to `UserService.AddUser`.
- **`UserChangedMessage` / `UserModel`** (`UserManagement/Messaging`, `UserManagement/Model`) — after a successful save, a `UserChangedMessage(UserChangeType.Added, userModel)` is sent through `WeakReferenceMessenger.Default` so the rest of the module (e.g. `ShowUsersViewModel`) can refresh.
- **Callers** — `UserViewModel` (composition host), the WPF `UserView.xaml` and WinForms `UserForm.cs` views (data binding), `ConsoleApp1/Program.cs` (drives `SaveUserAsync()` in a loop), `DesignAddUserViewModel` (`UserManagement/Design/DesignAddUserViewModel.cs`, a designer-time subclass), and `TestProject1/UserViewModelTest.cs`.

### Behavior & Details

**Form fields.** `_firstName` and `_lastName` use CommunityToolkit's `[ObservableProperty]`, which generates the public `FirstName`/`LastName` properties (satisfying `IUser`) and the change handlers. Each field is annotated `[NotifyCanExecuteChangedFor(nameof(SaveUserCommand))]`, so `SaveUserCommand` re-evaluates its `CanExecute` whenever a field changes. Each generated change handler calls `ValidateProperty(this, nameof(...))`, giving live per-field validation: an error is reported through `ErrorsChanged` as soon as a field becomes empty (which clears the WPF/WinForms input errors the moment the user starts typing).

Note that validation validates the view model itself, not a DTO — `ValidateProperty(this, ...)` passes the view model as the `IUser` to the validator, so the rules read the live `FirstName`/`LastName` properties.

**Saving (`SaveUserAsync`).** The source has an `[AsyncRelayCommand]`-style `[RelayCommand]` method, so the toolkit generates the `SaveUserCommand` async relay command named in the interface. The method:

1. Runs `ValidateAllProperties(this)`, which clears and repopulates the inherited error dictionary.
2. If `HasErrors`, shows `MessageService.ShowError("User data is invalid!")` and returns — no save attempt.
3. Otherwise awaits `UserService.AddUser(new UserDto { FirstName, LastName })`.
4. Sends `new UserChangedMessage(UserChangeType.Added, new UserModel { ... })` on `WeakReferenceMessenger.Default` so other parts of the app learn about the new user.
5. Resets `FirstName` and `LastName` to `string.Empty` so the form is ready for the next entry (the tests assert the fields are empty after a successful save).

**Exception handling.** Only `BusinessLayerException` is caught, and the handler is `throw;` — a bare rethrow. This means the catch adds no behaviour of its own: a business-layer failure (e.g. a duplicate user, if `UserService` raises one) propagates to the caller of `SaveUserAsync`/`SaveUserCommand`, and the fields are *not* reset when it does. Any other exception type likewise propagates.

### Rough edges

- The `catch (BusinessLayerException) { throw; }` block is inert — it neither logs nor swallows, so the catch has no observable effect.
- `LoggingService` is imported and declared on `IAddUserViewModel` but is never used in this class.
- `OnFirstNameChanged` contains a commented-out block that synced the value into a `_user` field; no such field exists, so it is dead code.
- The `[RelayCommand]` comment says the toolkit generates `SaveUserCommand` from the `Async` suffix; the attribute itself is just `[RelayCommand]`, so `SaveUserCommand` is an `IAsyncRelayCommand` and its `CanExecute` is re-queried via the `NotifyCanExecuteChangedFor` attributes rather than any explicit `CanSaveUser` predicate.