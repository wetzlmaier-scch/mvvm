# `BusinessLayer/UserService.cs`

`UserService` is the business-logic component for user management. It implements `IUserService` (defined in `BusinessLayer/IUserService.cs`) and provides two operations: `GetActiveUsers()` and `AddUser(UserDto)`.

### Purpose

This class sits between the `UserManagement` view-models and the NHibernate-based data access layer. It handles user queries and creates a new user when a valid `UserDto` is passed to it. When a save fails, it logs the error and wraps the exception in `BusinessLayerException` so callers only need to know one exception type for business failures.

### Role in the Codebase

The class lives in `BusinessLayer/` and is wired up via MEF. The `[Export(typeof(IUserService))]` attribute publishes it to the composition container; consumers (such as `ShowUsersViewModel` and `AddUserViewModel` in `UserManagement/ViewModel/`) import it via `[Import] public IUserService UserService`. The `[ImportingConstructor]` on the constructor tells MEF to inject `IRepositoryFactory`.

### Relationships

- **`IUserService`** (`BusinessLayer/IUserService.cs`) — the contract implemented here. It declares both operations plus a `LoggingService` property.
- **`IRepositoryFactory`** (`BusinessLayer/IRepositoryFactory.cs`) — injected via the constructor. This interface extends `IHibernateRepositoryFactory<int>` (`DataAccessLayer.Hibernate/IHibernateRepositoryFactory.cs`), whose `Create()` returns `IHibernateRepository<int>`.
- **`User`** (`Entity.Hibernate/User.cs`) — the NHibernate entity persisted on save.
- **`UserDto`** (`Dto/UserDto.cs`) — the data object used at the business-layer boundary.
- **`BusinessLayerException`** (`BusinessLayer/BusinessLayerException.cs`) — a simple `Exception` wrapper with `(message, inner)`.
- **`ILoggingService`** (`Logging/ILoggingService.cs`) — declared as a settable property on `IUserService` and implemented here. It is not injected via the constructor; the constructor sets it to `null!` and the composition container is expected to populate it.
- **Callers** — `ShowUsersViewModel` calls `GetActiveUsers()` on load; `AddUserViewModel` calls `AddUser()` from its save command and catches `BusinessLayerException`.

### Behavior & Details

**`GetActiveUsers()`** — Creates a repository, runs a LINQ query over `GetQueryable<User>()`, projects each entity to a `UserDto` (id, first name, last name only), and materializes it with `ToListAsync()`. The NHibernate `IsActive` filter is not applied in this query — the method name suggests it, but the query is unfiltered.

**`AddUser(UserDto user)`** —

1. Validates the input with `CommunityToolkit.Diagnostics` `Guard` — `FirstName` and `LastName` must be non-null and non-whitespace. These guards throw `ArgumentException`, which is **not** wrapped in `BusinessLayerException`.
2. Opens a repository and begins a unit-of-work transaction.
3. Saves a new `User` entity, then commits.
4. On any exception, rolls back the transaction, logs the error through `LoggingService`, and rethrows as `BusinessLayerException("Error saving user", e)`.

The comment in the code ("only throw a BusinessException") states the intent that callers should only ever see `BusinessLayerException`. In practice, validation failures bypass this because the `Guard` calls throw `ArgumentException` directly.

### Rough edges

- `GetActiveUsers()` does not filter by `IsActive`; it returns all users despite the method name.
- Validation failures in `AddUser` throw `ArgumentException` (from `Guard`), so the "callers only ever see `BusinessLayerException`" contract stated in the code comment is broken on that path.
- `LoggingService` is a public settable property set to `null!` in the constructor. If MEF does not inject a value before `AddUser` is called and an exception occurs, the log call will throw a `NullReferenceException`.