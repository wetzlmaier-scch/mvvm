# TestProject1/UserViewModelTest.cs

### Purpose
This class contains unit tests for the user management view-models in the `UserManagement` module — specifically `AddUserViewModel` and `ShowUsersViewModel`. It verifies the core user-management flows: adding a new user (including input validation and error handling) and loading the active users list into the view. It exists to give fast feedback on regressions in the user-management view-model layer without needing a real data access stack.

### Role in the Codebase
It is the only test class in the `TestProject1` project (alongside the project file and `MSTestSettings.cs`, which opts the assembly into method-level parallel execution). The project references `BusinessLayer` and `UserManagement` directly and tests the view-models against mocked services. If it were removed, nothing at runtime would break — it is pure test code — but the view-models' validation, error-propagation, and collection-population behaviour would go unverified.

Note the `[DoNotParallelize]` attribute: because `MSTestSettings.cs` enables parallel test execution, this class is explicitly excluded from it. That is meaningful, because the tests share a process-global side effect — `WeakReferenceMessenger.Default` — that cannot be isolated per test (see Rough edges).

### Relationships
What it depends on:

- `UserViewModel`, `AddUserViewModel`, `ShowUsersViewModel` (in `UserManagement/ViewModel/`) — the system under test. `UserViewModel` is a thin `MainViewModelBase` whose `AddUserViewModel` and `ShowUsersViewModel` properties are normally populated by MEF `[Import]` in the real app; the tests instead assign them via object initializers and inject mock services into those properties.
- `IUserService` (in `BusinessLayer/IUserService.cs`) — mocked via Moq. The tests stub `AddUser(UserDto)` and `GetActiveUsers()`.
- `IMessageService` (in `Mvvm/IMessageService.cs`) — mocked to observe error presentation via `ShowError`.
- `UserDto` (in `Dto/UserDto.cs`) and `UserModel` (in `UserManagement/Model/UserModel.cs`) — the DTO passed to the service and the observable model stored in `ShowUsersViewModel.Users` respectively.
- `BusinessLayerException` (in `BusinessLayer/BusinessLayerException.cs`) — the exception type asserted to propagate out of `SaveUserAsync`.
- `WeakReferenceMessenger.Default` (CommunityToolkit.Mvvm) — cleaned up in `Teardown`.
- Testing stack: MSTest (`[TestClass]`, `[TestMethod]`, `[DataRow]`, `[TestInitialize]`, `[TestCleanup]`, `Assert`), Moq.

Nothing in the repository calls this class; it is discovered by the test runner.

### Behavior & Details
`Setup` builds the same arrangement for every test: two Moq mocks (`IUserService`, `IMessageService`) and a fresh `UserViewModel` with real `ShowUsersViewModel` and `AddUserViewModel` instances wired to those mocks. Because `ShowUsersViewModel`'s constructor subscribes itself to `WeakReferenceMessenger` for `UserChangedMessage`, `Teardown` calls `WeakReferenceMessenger.Default.UnregisterAll` on both child view-models to clear those global subscriptions between tests.

The four test methods:

- `CanSaveUser_ValidatesInputsCorrectly` — data-driven (`[DataRow]`) over four invalid input pairs: empty and whitespace-only `FirstName`/`LastName`. It calls `SaveUserAsync` and verifies the validation-failure path in `AddUserViewModel`: the error message `"User data is invalid!"` is shown once through `MessageService` and the service receives no calls at all (`VerifyNoOtherCalls`). The service stub returning `Task.FromResult(1)` is set up defensively but is never reached.
- `SaveUserAsync_WhenSuccessful_AddsUserToCollectionAndClearsFields` — the happy path. With valid `FirstName`/`LastName`, it verifies (1) `UserService.AddUser` was called once with a `UserDto` carrying those exact names, (2) `ShowUsersViewModel.Users` now contains one entry with those names, and (3) the input fields were reset to `string.Empty`. The added entry is a `UserModel` that arrived via the `UserChangedMessage` broadcast sent by `SaveUserAsync`, handled by `ShowUsersViewModel`'s messenger registration.
- `SaveUserAsync_WhenServiceThrows_RethrowsExceptionAndDoesNotAddUser` — stubs `AddUser` to throw a `BusinessLayerException` and drives the flow through the generated `SaveUserCommand` (an `IAsyncRelayCommand`) rather than calling `SaveUserAsync` directly. It asserts the exception is rethrown with its original message and that `Users` stays empty — i.e. the failure leaves no partial UI state.
- `OnLoadedAsync_FetchesActiveUsers_AndPopulatesCollection` — stubs `GetActiveUsers` with two users and invokes `ShowUsersViewModel.LoadedCommand`. Verifies the service call happened once and that `Users` was populated in order with the returned data (converted from `UserDto` to `UserModel`).

### Rough edges
- `Teardown` does not unregister `_viewModel` itself, even though `UserViewModel` extends `MainViewModelBase`, which may also subscribe to the messenger. Each test therefore accumulates a dead registration on the shared `WeakReferenceMessenger.Default` for the whole run.
- The success and validation tests call `SaveUserAsync()` directly, while the exception test goes through `SaveUserCommand.ExecuteAsync(null)` (and a commented-out line references a non-existent `SaveUserCommand` on `UserViewModel`). Since the command is generated from the same method, coverage of the command wrapper is incidental and inconsistent across tests.
- The assertion that a new user reaches `ShowUsersViewModel.Users` couples the test to the `WeakReferenceMessenger` broadcast in `SaveUserAsync` (which the test neither registers for nor unregisters from); that global messenger is also why the class carries `[DoNotParallelize]` despite the assembly opting into parallel execution.