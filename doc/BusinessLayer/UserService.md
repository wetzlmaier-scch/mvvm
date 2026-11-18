# BusinessLayer/UserService.cs

### Purpose

`UserService` is the business-logic component for user management in this MEF-composed application. It implements `IUserService` (defined in `BusinessLayer/IUserService.cs`) and provides two operations: listing users (`GetActiveUsers`) and creating a user (`AddUser`). It exists so the view-model layer can talk to the persistence layer through a stable interface without knowing NHibernate or the repository types.

### Role in the codebase

The class lives in `BusinessLayer/`, sits between the `UserManagement` view-models and the NHibernate-based data access layer, and is wired up through MEF: `[Export(typeof(IUserService))]` publishes it, and `[ImportingConstructor]` takes its one dependency, `IRepositoryFactory` (implemented by `RepositoryFactory` in the same folder). If it were removed, `AddUserViewModel` and `ShowUsersViewModel` could no longer load or create users, and MEF composition would fail to satisfy their `[Import]` of `IUserService`.

### Relationships

- **`IUserService`** (`BusinessLayer/IUserService.cs`) — the contract this class implements, also declaring the `LoggingService` property.
- **`IRepositoryFactory`** (`BusinessLayer/IRepositoryFactory.cs`) — inherits `IHibernateRepositoryFactory<int>` from `DataAccessLayer.Hibernate`, which returns an `IHibernateRepository<int>` (NHibernate, `IDisposable`) from `Create()`. `UserService` creates one repository per operation and disposes it with `using`.
- **`User`** (`Entity.Hibernate/User.cs`) — the NHibernate entity; its public `User(firstName, lastName)` constructor also sets `IsActive = true`.
- **`UserDto`** (from the `Dto` namespace) — the shape crossing the layer boundary; `UserDto` and `User` share `Id`, `FirstName`, `LastName`.
- **`ILoggingService`** (`Logging/ILoggingService.cs`) — used to log save failures.
- **`BusinessLayerException`** (`BusinessLayer/BusinessLayerException.cs`) — a simple `Exception` wrapper with `(message, inner)`.
- **Callers** — `UserManagement/ViewModel/ShowUsersViewModel.cs` calls `GetActiveUsers()` on load; `AddUserViewModel.cs` calls `AddUser()` from its save command and catches `BusinessLayerException`. `TestProject1/UserViewModelTest.cs` mocks `IUserService` rather than using this class directly.

### Behavior & Details

**`GetActiveUsers()`** — Creates a repository, runs a LINQ query over `GetQueryable<User>()`, projects each entity to a `UserDto` (id, first name, last name only), and materializes it with `ToListAsync()`. Despite the name, the query contains no `IsActive` filter, so it returns every user in the table, not only active ones.

**`AddUser(UserDto user)`** — Runs in three steps:

1. Validates the input with `CommunityToolkit.Diagnostics` `Guard` — `FirstName` and `LastName` must be non-null and non-whitespace. These guards throw a generic `ArgumentException` (not `BusinessLayerException`), so callers cannot distinguish bad input from save failures by exception type.
2. Opens a transaction via `repository.UnitOfWork.BeginTransaction()` and saves a new `User` entity.
3. On success commits; on any exception it rolls back, logs the error through `LoggingService`, and rethrows as `BusinessLayerException("Error saving user", e)`. The comment in the code states this is the only exception type callers should ever see from this service.

The `LoggingService` property is public with a setter (part of `IUserService`) and is meant to be injected by the composition container rather than the constructor. Note that in the constructor it is set to `null!` and the property carries no `[Import]` attribute, so whether it is ever populated depends entirely on the host's MEF configuration. If the property is not set, `AddUser` will throw a `NullReferenceException` inside its own `catch` block when it tries to log, masking the original error.

### Rough edges

- `GetActiveUsers` does not filter by `IsActive`; it returns all users, contradicting its name.
- `LoggingService` is not MEF-injected by this class (no `[Import]`); anything relying on the error path must guarantee the property is set externally, or a save failure becomes a `NullReferenceException` from within the service's own `catch`.
- `AddUser`'s `Guard` calls throw `ArgumentException`, so validation failures bypass the "callers only ever see `BusinessLayerException`" contract stated in the code.