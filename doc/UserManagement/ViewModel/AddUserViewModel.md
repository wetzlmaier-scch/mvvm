# `AddUserViewModel` (`UserManagement/ViewModel/AddUserViewModel.cs`)

### Purpose

This class is the view model behind the "create a new user" form. It holds the two form fields (`FirstName`, `LastName`), validates them as the user types, persists a new user through the business layer, and then announces the addition to the rest of the application so the user list stays in sync.

### Role in the Codebase

It is a leaf view model in the `UserManagement/ViewModel` module, paired with its sibling `ShowUsersViewModel` (the user-list side). The parent `UserViewModel` does not create it directly — both are MEF exports: the class is published via `[Export(typeof(IAddUserViewModel))]` and injected into `UserViewModel` through its `[Import]` property. UI (the WPF `UserView.xaml` and WinForms `UserForm`) binds to this class through the `IAddUserViewModel` interface, the `ConsoleApp1/Program.cs` entry point drives it headlessly, and `TestProject1/UserViewModelTest.cs` instantiates it for unit tests. `Design/DesignAddUserViewModel.cs` subclasses it to feed static design data to the views in the designer. Removing it would break user creation in every front end.

### Relationships

- **`IAddUserViewModel`** (`UserManagement/ViewModel/IAddUserViewModel.cs`) — the interface this class implements. It extends `INotifyDataErrorInfo` and `IUser`, so views bind validation state and this class can be handed straight to the validator.
- **`ValidatableViewModelBase<T>`** (`Mvvm/ValidatableViewModelBase.cs`, in-repo) — the base class. This class passes `new UserValidator()` as the `IValidator<IUser>`; the base class implements `INotifyDataErrorInfo` (error dictionary, `HasErrors`, `ErrorsChanged`) and provides the `ValidateAllProperties` / `ValidateProperty` helpers. Inherited up the chain: `ViewModelBase` → `ObservableObject`.
- **`UserValidator`** (`Dto/Validation/UserValidator.cs`) — FluentValidation rules: `FirstName` and `LastName` must be non-empty ("Firstname is required." / "Lastname is required.").
- **`IUserService`** (defined in `BusinessLayer/IUserService.cs`, implemented by `BusinessLayer/UserService.cs`) — MEF-injected; `AddUser(UserDto)` is the persistence call.
- **`IMessageService`** (Logging project, imported) — shows the "User data is invalid!" error popup when validation fails.
- **`UserModel` / `UserChangedMessage` / `UserChangeType`** (`UserManagement/Model`, `UserManagement/Messaging`) — sent over `WeakReferenceMessenger` after a successful save; `ShowUsersViewModel` subscribes in its constructor and appends the new user to its list.
- **`IUser`, `UserDto`** (`Dto` project) — `IUser` is the shape the validator runs against (this class implements it via the interface); `UserDto` is the payload handed to the service.

### Behavior & Details

**Fields and live validation.** `FirstName` and `LastName` are generated from `_firstName` / `_lastName` by the CommunityToolkit.Mvvm `[ObservableProperty]` source generator. Each change fires its `On...Changed` partial, which calls `ValidateProperty(this, nameof(...))` from the base class, so a single field is re-validated on every keystroke. The same attributes mark the properties with `[NotifyCanExecuteChangedFor(nameof(SaveUserCommand))]`, which re-evaluates the command's ability to run whenever a field changes (in practice, `SaveUserCommand` has no explicit guard, so this mainly matters if one is added). The WPF view binds the text boxes with `ValidatesOnNotifyDataErrors=True`, so the base class's `INotifyDataErrorInfo` errors surface as red validation borders.

**Saving (`SaveUserAsync`, exposed as `SaveUserCommand`).** The source generator strips the `Async` suffix and produces the `SaveUserCommand` async relay command. The flow:

1. `ValidateAllProperties(this)` re-runs the full `UserValidator` and clears the base class's error dictionary first, so the save decision is based on current field values, not stale errors.
2. If `HasErrors`, it shows the "User data is invalid!" message via `MessageService` and returns — nothing is saved.
3. Otherwise it calls `UserService.AddUser(new UserDto { FirstName, LastName })`. That service re-checks both fields with `Guard` and wraps any failure (roll back transaction, log, rethrow as `BusinessLayerException`).
4. On success it broadcasts `new UserChangedMessage(UserChangeType.Added, userModel)` over `WeakReferenceMessenger.Default` — this is the only coupling to the list side; it never touches `ShowUsersViewModel`'s collection directly.
5. It then resets `FirstName` and `LastName` to `string.Empty`, so the form is ready for the next user. Note this re-triggers the per-field validation, which re-registers the "required" errors for the now-empty fields.
6. A `catch (BusinessLayerException) { throw; }` deliberately rethrows business failures so they propagate to the command's caller (in WPF, that means the `IAsyncRelayCommand` surfaces the exception).

### Rough edges

- The `catch (BusinessLayerException) { throw; }` block is a no-op — it rethrows exactly as it would without the catch. It documents intent (business errors are expected to bubble) but has no observable effect.
- `UserService.AddUser` runs `Guard` checks *before* its try block, so invalid input throws a plain `ArgumentException` rather than the `BusinessLayerException` the comment promises "callers only ever see". This class's business-error catch therefore won't catch that path.
- The `OnFirstNameChanged` body is fully commented out — the old sync-to-`_user` logic remains as dead code.
- `LoggingService` is imported but never used anywhere in this class.
- After a successful save the cleared fields immediately re-report "required" errors, so the empty form shows as invalid until the user types again.