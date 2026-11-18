# `TestProject1/UserViewModelTest.cs`

### Purpose

This is the test class `UserViewModelTest` — the only test in the repository. It verifies the behavior of the user-management MVVM stack: that adding a user validates input, persists through `IUserService`, and updates the in-memory user list, and that loading active users populates that list. The class is marked `[DoNotParallelize]` even though `MSTestSettings.cs` enables method-level parallel assembly-wide, and `TestProject1.csproj` references the `BusinessLayer` and `UserManagement` projects — the class under test is the WPF-tied `UserViewModel`, so tests are forced to run serially in one process.

### Role in the Codebase

`TestProject1` holds exactly one test file, so this class is the project's entire test suite. It guards the contract between the view models in `UserManagement/ViewModel/` and the business layer: anyone changing `SaveUserAsync`, `AddUserViewModel` validation, `ShowUsersViewModel`'s user collection, or `IUserService`'s signatures has a failing test to point at. There is no other automated coverage in the repository for this feature.

### Relationships

- **System under test:** `UserViewModel` (`UserManagement/ViewModel/UserViewModel.cs`). It is a composition container (MEF `[Export]`/`[Import]`) holding `AddUserViewModel` and `ShowUsersViewModel` as `[Import]` properties. The tests bypass MEF and assign those properties directly, which works because they are plain settable properties.
- **Mocked collaborators:** `IUserService` (`BusinessLayer/IUserService.cs`, interface for `GetActiveUsers()` and `AddUser(UserDto)`) and `IMessageService` (`Mvvm/IMessageService.cs`, a UI message-box abstraction). Both are mocked with Moq; `IUserService.LoggingService` is left unstubbed and never touched by the paths under test.
- **Transitive dependencies worth knowing:**
  - `AddUserViewModel` (`UserManagement/ViewModel/AddUserViewModel.cs`) extends `ValidatableViewModelBase<IUser>` with `UserValidator` (`Dto/Validation/UserValidator.cs`) as its FluentValidation validator. Its `UserValidator` rules are `NotEmpty` only — a whitespace-only name like `"   "` passes validation, so whitespace-only rejection (asserted in one test) would come from `SaveUserAsync` or a trim the view model performs; that logic is not visible in the test itself. Its constructor takes the validator, so the real `UserValidator` runs inside the view model.
  - `AddUserViewModel` also has an `[Import] ILoggingService` that the setup does not assign; the code paths tested never call it.
  - `ShowUsersViewModel` (`UserManagement/ViewModel/ShowUsersViewModel.cs`) registers itself with `WeakReferenceMessenger` for `UserChangedMessage` in its constructor; `AddUserViewModel.SaveUserAsync` sends `UserChangedMessage(Added, ...)` after a successful save. The `Teardown` call to `WeakReferenceMessenger.Default.UnregisterAll(...)` on both child view models exists because of this registration — the messenger is a process-wide static singleton, so a leak between tests would show up as duplicate entries in `Users`.
  - `UserViewModel` extends `MainViewModelBase` (`Mvvm/MainViewModelBase.cs`), whose constructor reaches into `Application.Current?.MainWindow` — this is what forces the test project to target `net10.0-windows` and makes the WPF assembly available without a running window.
  - Data shapes: `UserDto` (`Dto/UserDto.cs`) is what the view model hands to `IUserService`; `UserModel` (`UserManagement/Model/UserModel.cs`) is what ends up in `ShowUsersViewModel.Users`.

### Behavior & Details

`Setup` builds a fresh `UserViewModel` per test with the two child view models wired to shared mock objects: `UserService` is the same `Mock<IUserService>.Object` for both children, so a single stub covers save-and-load flows.

- `CanSaveUser_ValidatesInputsCorrectly` — table-driven over four pairs: empty first name, empty last name, whitespace first name, whitespace last name. Stubs `AddUser` to succeed and asserts that `ShowError("User data is invalid!")` fires exactly once and no other `IUserService` method is reached. This documents the contract that invalid input never touches the service. The whitespace rows pin down trimming/whitespace handling that the `NotEmpty` rules in `UserValidator` alone would not enforce.
- `SaveUserAsync_WhenSuccessful_AddsUserToCollectionAndClearsFields` — the happy path. After `SaveUserAsync`, it verifies: (1) `AddUser` received a `UserDto` with `FirstName == "Jane"` and `LastName == "Smith"`; (2) `ShowUsersViewModel.Users` now contains exactly one `UserModel` with those values — this entry arrives via the `WeakReferenceMessenger` `UserChangedMessage(Added, ...)` that `SaveUserAsync` sends, so the test also implicitly covers the messenger plumbing between the two child view models; (3) `FirstName` and `LastName` were reset to `string.Empty`. A commented-out call to `SaveUserCommand.ExecuteAsync(null)` indicates the test used to drive the generated `IAsyncRelayCommand` and was switched to calling `SaveUserAsync` directly.
- `SaveUserAsync_WhenServiceThrows_RethrowsExceptionAndDoesNotAddUser` — makes `AddUser` throw `BusinessLayerException` (`BusinessLayer/BusinessLayerException.cs`, a plain `Exception` subclass) and asserts it propagates out of `SaveUserCommand.ExecuteAsync(null)` (this test does use the command, so the `NotifyCanExecuteChangedFor`-wired command path is exercised at least once) and that `Users` is untouched. Note the contrast with the validation test: validation failures swallow the input into an error message; service failures surface as exceptions.
- `OnLoadedAsync_FetchesActiveUsers_AndPopulatesCollection` — stubs `GetActiveUsers` with two `UserDto`s and drives `ShowUsersViewModel.LoadedCommand.ExecuteAsync(null)`, asserting the service was called once and `Users` contains both entries in order as `UserModel`s. This also doubles as a regression check that the `UserChangedMessage` registration from the previous tests did not leak (the count is exactly 2, not 3).

No test covers `IMessageService.ShowMessage`/`ShowWarning`, `UserChangedMessage.Updated`/`Deleted` (the view model throws `NotImplementedException` for both), or the `IMainViewModel`/window behavior of `UserViewModel` itself — those paths are untested by design of this suite.

### Rough edges

- The whitespace cases in `CanSaveUser_ValidatesInputsCorrectly` assert rejection, but `UserValidator` only applies `NotEmpty` — the trim/whitespace rule that makes those rows pass is not visible from this test file, so the test documents behavior without making its source obvious.
- The setup never assigns `LoggingService` on `AddUserViewModel` or `ShowUsersViewModel`; any future change that makes those paths call `LoggingService` will produce a null-reference failure that is not about the behavior under test.
- The class is sealed and `[DoNotParallelize]`, but only `ShowUsersViewModel` actually registers with the global messenger, and nothing in this test file touches WPF application state — the parallelization guard appears broader than what the visible tests require.