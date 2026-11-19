# BusinessLayer/IUserService.cs

### Purpose

`IUserService` is the business-layer contract for user management in this solution. It exists so that the `UserManagement` view models (and the unit tests) can depend on an abstraction for querying and creating users instead of on `UserService` directly. It declares exactly two operations — fetch the active users, add a new user — plus a settable `LoggingService` property.

### Role in the Codebase

The interface is the seam between the MVVM `UserManagement` module and the `BusinessLayer` project. It is the type that flows through MEF: consumers declare `[Import] public IUserService UserService { get; set; }` and the composition container fills it with the sole implementation, `UserService` (`BusinessLayer/UserService.cs`), which publishes itself via `[Export(typeof(IUserService))]`. Removing the interface would force every view model to bind to the concrete `UserService` and would break the mocked stubs in `TestProject1`.

### Relationships

- **`UserDto`** (`Dto/UserDto.cs`) — the data shape both methods exchange. A small POCO with `Id`, `FirstName`, `LastName`; it implements the `Dto.IUser` contract.
- **`ILoggingService`** (`Logging/ILoggingService.cs`) — declared as a settable property on the interface, implemented in the `Logging` project. It extends `Microsoft.Extensions.Logging.ILogger` with strongly-typed overloads such as `LogError(Exception?, string?, ...)`.
- **Consumers:**
  - `UserManagement/ViewModel/ShowUsersViewModel.cs` — imports the service and calls `GetActiveUsers()` in `OnLoadedAsync()` to populate its `Users` collection.
  - `UserManagement/ViewModel/AddUserViewModel.cs` — imports the service and calls `AddUser(UserDto)` from `SaveUserAsync()` after its own validation passes.
  - `TestProject1/UserViewModelTest.cs` — builds a `Mock<IUserService>` to stub both operations.

### Behavior & Details

- `Task<List<UserDto>> GetActiveUsers()` — contract for loading all currently active users, projected into DTOs. The implementation reads through NHibernate and never throws a checked exception of its own; callers treat a faulting call as an environment failure.
- `Task AddUser(UserDto user)` — contract for persisting a new user. Two things a caller must know, both enforced by the implementation rather than the contract:
  - `user.FirstName` and `user.LastName` must be non-null and non-whitespace, or a `CommunityToolkit` `ArgumentException` is thrown *before* any database work.
  - Any persistence failure is logged via the service's `LoggingService` and rethrown as `BusinessLayerException`. `AddUserViewModel.SaveUserAsync` deliberately rethrows `BusinessLayerException` uncaught, so it propagates up to the composition/host layer.
- `ILoggingService LoggingService { get; set; }` — a settable property on the interface, not a method parameter. It is part of the contract because `UserService.AddUser` calls `LoggingService.LogError(...)` on failure and would hit a `NullReferenceException` if never assigned. The concrete implementation initializes it to `null!` in its constructor, so the property is only usable once MEF property injection has completed. Callers of the interface should treat it as "set before first use by the container," not something they assign themselves.

### Rough edges

The `LoggingService` property is unusual in the interface: it is business-layer plumbing (the service needs a logger for `AddUser`'s failure path) but it is exposed on the public contract and has no injection annotation of its own — MEF fills it only because the `UserService` implementation relies on the container to set it after construction. A caller who constructs or fakes `IUserService` outside MEF gets a property that must be assigned manually before any `AddUser` failure can be logged.