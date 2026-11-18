### Purpose

`TestProject1/UserViewModelTest.cs` is the unit-test suite for the user-management ViewModels in the `UserManagement` project. It defines one MSTest class, `UserViewModelTest`, which exercises two pieces of behaviour in the real `UserViewModel` object graph: adding a user (with input validation and error surfacing, in `AddUserViewModel`) and loading the active user list (in `ShowUsersViewModel`). The `IUserService` and `IMessageService` dependencies are mocked with Moq so the tests verify the ViewModels' orchestration logic without touching a database or showing real dialogs.

### Role in the Codebase

The `TestProject1` project is the solution's test project (MSTest on `net10.0-windows`, Moq, referencing `BusinessLayer` and `UserManagement`). This class is its only test. The ViewModels under test are `MefExported` composition parts — normally wired up by MEF imports at runtime — so the tests construct them by hand and assign the `UserService`/`MessageService` properties directly, which is why the properties exist as settable `[Import]` properties in the source. Removing this file loses the coverage of the add-user validation path, the successful add path (collection update + field clearing), and the initial load path.

### Relationships

- **`UserViewModel`** (`UserManagement/ViewModel/UserViewModel.cs`): the object under test. It is a `MainViewModelBase` that exposes `AddUserViewModel` and `ShowUsersViewModel` (MefImported in the app; instantiated manually here).
- **`AddUserViewModel`** (`UserManagement/ViewModel/AddUserViewModel.cs`): derives from `ValidatableViewModelBase<IUser>` with a `UserValidator` (FluentValidation rule: `FirstName` and `LastName` must be non-empty). `SaveUserAsync()` — exposed as `SaveUserCommand` via `[RelayCommand]` — validates, calls `MessageService.ShowError("User data is invalid!")` if invalid, otherwise calls `UserService.AddUser`, sends a `UserChangedMessage` over `WeakReferenceMessenger`, and clears the name fields.
- **`ShowUsersViewModel`** (`UserManagement/ViewModel/ShowUsersViewModel.cs`): registers for `UserChangedMessage` in its constructor (an `Added` message appends to `Users`); `OnLoadedAsync` — exposed as `LoadedCommand` — fetches `UserService.GetActiveUsers()` and populates its `Users` collection.
- **`IUserService`** (`BusinessLayer/IUserService.cs`): mocked; provides `GetActiveUsers()` and `AddUser(UserDto)`.
- **`IMessageService`** (`Mvvm/IMessageService.cs`): mocked; the tests verify the exact `ShowError` call.
- **`UserChangedMessage` / `WeakReferenceMessenger`** (`UserManagement.Messaging`, CommunityToolkit.Mvvm): the messaging channel that makes `AddUserViewModel` and `ShowUsersViewModel` cooperate.

### Behavior & Details

**Test setup and isolation.** `[DoNotParallelize]` on the class keeps all four tests in the same class from running concurrently (MSTest otherwise parallelizes at method level per the assembly-level `Parallelize` attribute in `MSTestSettings.cs`). This matters because `Teardown` calls `WeakReferenceMessenger.Default.UnregisterAll(...)` on the shared static messenger for both child ViewModels; if tests ran in parallel, one test's teardown could unregister the other test's ViewModels and silently break its messaging assertions.

**`CanSaveUser_ValidatesInputsCorrectly`** — data-driven over four cases: empty first name, empty last name, and both whitespace-only variants. It sets the fields, stubs `AddUser` and `SaveUserAsync`, and asserts that `MessageService.ShowError("User data is invalid!")` is called once and the service is never called. Note that the whitespace cases only pass because the fields go through the `[ObservableProperty]` generated properties, whose change notifications (with their `NotifyCanExecuteChangedFor`/validation hooks) make validation treat `"   "` as invalid; in `ValidatableViewModelBase` the error is tracked per property, and `ValidateAllProperties` inside `SaveUserAsync` runs the FluentValidation `NotEmpty` rule against the current property value.

**`SaveUserAsync_WhenSuccessful_AddsUserToCollectionAndClearsFields`** — the happy path. After calling `SaveUserAsync`, it verifies three effects: the service received a `UserDto` with the exact names; the user appears in `ShowUsersViewModel.Users` (this happens through the `UserChangedMessage` posted by `SaveUserAsync` and handled by `ShowUsersViewModel`'s constructor-time messenger registration — so the test implicitly covers the messaging contract); and `FirstName`/`LastName` are reset to `string.Empty`.

**`SaveUserAsync_WhenServiceThrows_RethrowsExceptionAndDoesNotAddUser`** — `AddUser` is stubbed to throw a `BusinessLayerException` (with an inner `Exception`, mirroring the catch clause in `SaveUserAsync` that rethrows only `BusinessLayerException`). It asserts the exception propagates out of the command unchanged, with its message, and that `Users` remains empty — i.e., the failure path neither swallows the error nor leaves stale UI state.

**`OnLoadedAsync_FetchesActiveUsers_AndPopulatesCollection`** — stubs `GetActiveUsers` with two `UserDto`s, executes `LoadedCommand`, and asserts the service was called once and `Users` contains both entries in order. It does not exercise the `UserChangedMessage` path (messages sent by `SaveUserAsync` are the other way around).

### Rough edges

- The `UserService` mock in the validation test stubs `AddUser` although the method under test never reaches it — harmless but misleading; `VerifyNoOtherCalls` is the real assertion.
- In `SaveUserAsync_WhenSuccessful_...`, the comment says `//await _viewModel.SaveUserCommand.ExecuteAsync(null);` — a commented-out call on `UserViewModel`, which has no such command (it lives on `AddUserViewModel`). It is dead code that could confuse a future reader.
- The whitespace `[DataRow]` cases depend on the generated property setters doing their notification/validation dance; if `SaveUserAsync` were refactored to validate a different object, or the properties changed shape, these cases could behave differently than the test names imply.