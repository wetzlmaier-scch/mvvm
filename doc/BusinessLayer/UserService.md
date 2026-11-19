`BusinessLayer/UserService.cs`

### Purpose
`UserService` is the business-layer service for user management. It provides the two operations the application's user features need: listing active users and creating a new one. It is the concrete implementation of `IUserService` (defined in `BusinessLayer/IUserService.cs`), and it's wired into the application as a MEF component via `[Export(typeof(IUserService))]`, so consumers receive it through the interface without knowing this class exists.

### Role in the Codebase
This class sits in `BusinessLayer/`, which holds the application's business logic as MEF-exported services behind interfaces. `UserService` is one such service; the MEF container composes it with `IRepositoryFactory` (and, for callers that need logging, an `ILoggingService`).

The concrete callers that depend on it are in the WPF `UserManagement` project: `UserManagement/ViewModel/ShowUsersViewModel.cs` and `UserManagement/ViewModel/AddUserViewModel.cs` both hold a settable `IUserService` property (presumably populated by MEF via `[Import]`-style composition), and `TestProject1/UserViewModelTest.cs` mocks `IUserService` to test those view models. If this class were removed, the MEF container would have nothing to satisfy the `IUserService` export, and the user view models in `UserManagement` would have no real backend.

### Relationships
- **`IUserService`** (`BusinessLayer/IUserService.cs`): the contract this class implements — `GetActiveUsers()`, `AddUser(UserDto)`, and a `LoggingService` property.
- **`IRepositoryFactory`** (`BusinessLayer/IRepositoryFactory.cs`): injected through the `[ImportingConstructor]`. This marker interface extends `IHibernateRepositoryFactory<int>` (in `DataAccessLayer.Hibernate`), whose concrete implementation `HibernateRepositoryFactory<TKey>` builds an NHibernate session factory and hands out one repository per `Create()` call.
- **`User`** (`Entity.Hibernate/User.cs`): the NHibernate entity; `AddUser` builds one from the DTO. `User`'s constructor sets `IsActive = true`, which is why newly created users appear in `GetActiveUsers` results.
- **`UserDto`** (`Dto/UserDto.cs`): the DTO shape crossing the layer boundary, with `Id`, `FirstName`, `LastName`.
- **`BusinessLayerException`** (`BusinessLayer/BusinessLayerException.cs`): a plain `Exception` subclass used to wrap failures in `AddUser`.
- **`ILoggingService` / `LoggingService`** (`Logging/`): optional dependency exposed as a settable property. `LoggingService` is also a MEF export, so the container can inject it if the consuming component declares the import.

### Behavior & Details
**`GetActiveUsers`** opens a repository, queries all `User` entities, and projects them into `List<UserDto>` via LINQ-to-NHibernate, mapping only `Id`, `FirstName`, `LastName`. Despite its name, it does not filter on `IsActive` — it returns every user. The repository is `IDisposable`, so the `using` statement closes the underlying NHibernate session when the method completes.

**`AddUser`** follows a validate–transact–wrap pattern:

1. `CommunityToolkit.Diagnostics.Guard` checks that `user.FirstName` and `user.LastName` are not null or whitespace; a guard failure throws an `ArgumentException` immediately, before any repository is opened.
2. A repository is created and a transaction started via `repository.UnitOfWork.BeginTransaction()`.
3. Inside the transaction, a new `User` entity is saved (an NHibernate `Insert`), then the transaction is committed.
4. If anything throws, the catch block rolls the transaction back, logs the error through `LoggingService.LogError`, and rethrows as a `BusinessLayerException` so callers only ever see that one exception type.

**The `LoggingService` property** is `get; set;` with no backing logic — it exists purely so the MEF container can inject a logging implementation into the service. However, the constructor sets it to `null!`, and nothing else in this class assigns it. So in `AddUser`, the `LoggingService.LogError(...)` call will throw a `NullReferenceException` whenever the catch block runs — unless a consumer happens to assign the property after construction. Note that a `null!` suppression at the constructor means the failure surfaces at the log call, not at construction.

### Rough edges
- `GetActiveUsers` returns all users, not only active ones, contrary to its name; there is no `IsActive` filter.
- `AddUser` unconditionally calls `LoggingService.LogError`, but `LoggingService` starts as `null` and is never set here, so the rollback path crashes with a `NullReferenceException` unless the property is externally assigned.
- `AddUser` does not map `UserDto` validation failures (guard exceptions) into `BusinessLayerException`, only database failures — so the "callers only ever see `BusinessLayerException`" contract holds only for the save step.