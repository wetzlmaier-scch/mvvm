# BusinessLayer/IUserService.cs

### Purpose

`IUserService` is the business-layer contract for user management in this solution. It exists so that the `UserManagement` view models (and the unit tests) can depend on an abstraction for querying and creating users, rather than on the concrete `UserService`. The concrete implementation is `UserService` (`BusinessLayer/UserService.cs`), which is published to the MEF composition container via `[Export(typeof(IUserService))]`.

### Role in the Codebase

The interface is the seam between the MVVM `UserManagement` module and the `BusinessLayer` project. It is the type that flows through MEF: consumers declare a `[Import] public IUserService UserService` property and call through the interface. The contract is implemented by exactly one class, `UserService`, and is also the type mocked in `TestProject1/UserViewModelTest.cs` via Moq.

Removing the interface would break the MEF wiring (the `[Export]` on `UserService` and the `[Import]` properties on the view models all reference it) and the test suite's ability to substitute a fake.

### Relationships

- **`UserDto`** (`Dto/UserDto.cs`) — the data shape crossing this boundary. `GetActiveUsers()` returns `List<UserDto>`; `AddUser` takes one. `UserDto` implements `Dto.IUser` and adds `Id`; note the service layer deliberately uses the concrete DTO at its boundary rather than `IUser`.
- **`ILoggingService`** (`Logging/ILoggingService.cs`) — declared as a settable property on the interface, not as a constructor parameter. `UserService` inherits the full `Microsoft.Extensions.Logging.ILogger` surface plus its `Log*` overloads through this type and uses `LogError(e, "Error saving user")` in its save path.
- **Callers** — `UserManagement/ViewModel/ShowUsersViewModel.cs` calls `GetActiveUsers()` when the view loads; `UserManagement/ViewModel/AddUserViewModel.cs` calls `AddUser()`. `TestProject1/UserViewModelTest.cs` mocks the interface with Moq to test both view models.

### Behavior & Details

The interface has two operations plus one dependency property:

- **`Task<List<UserDto>> GetActiveUsers()`** — returns the current user list. The implementation queries NHibernate over the `User` entity and projects each to a `UserDto`.
- **`Task AddUser(UserDto user)`** — persists a new user. The implementation validates that `FirstName` and `LastName` are not null or whitespace, saves within an NHibernate transaction, rolls back on failure, logs the error through `LoggingService`, and rethrows as `BusinessLayerException` so callers only ever see that one exception type.
- **`ILoggingService LoggingService { get; set; }`** — a mutable dependency held on the contract itself. In `UserService` it is initialized to `null!` in the `[ImportingConstructor]` and is expected to be populated by the composition container (property import) before any method runs. Callers of the interface are not expected to set it; tests that use the interface directly will see a null `LoggingService` unless they assign one.

### Rough edges

The `LoggingService` property is declared on the interface and only used by the implementation, not by any caller — it couples the public contract to an internal implementation concern. Any future implementer must set this property to avoid a `NullReferenceException` on `AddUser`'s error path, even if it never logs.