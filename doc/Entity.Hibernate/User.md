### Purpose

`Entity.Hibernate/User.cs` defines the `User` domain entity — the NHibernate-mapped representation of an application user with a first name, last name, and active flag. It exists so that `BusinessLayer/UserService.cs` can create and persist users, and queries can retrieve them, with the mapping to the database handled by the separate `UserMapping.cs` in the same folder.

### Role in the Codebase

This class is the persistent domain model for the small user-management part of the app. It lives in the `Entity.Hibernate` project, which is referenced by `BusinessLayer` (see `BusinessLayer/BusinessLayer.csproj`), and `BusinessLayer/ModelContext.cs` tells NHibernate to discover mappings from that assembly so `UserMapping` is picked up. `UserManagement/ViewModel/ShowUsersViewModel.cs` imports the `Entity.Hibernate` namespace as well, though it works mainly through DTOs rather than the entity itself. Removing `User` would break `UserService.AddUser`, which constructs one directly, and all NHibernate persistence of users.

### Relationships

- **`DataAccessLayer.Entity<int>`** (defined in `DataAccessLayer/Entity.cs`) — the abstract base this class extends. It contributes the `Id` property (`int`, settable only within the class hierarchy, never outside) and implements `IEntity<int>` (defined in `DataAccessLayer/IEntity.cs`). All identity-related behavior is inherited; nothing about it is overridden here.
- **`Entity.Hibernate/UserMapping.cs`** — a Fluent NHibernate `ClassMap<User>` that maps this class to the `Users` table. `Id` is a database identity column, `FirstName` and `LastName` are non-nullable columns of length 50, and `IsActive` is a non-nullable bit. This is why the properties are declared `virtual` — NHibernate needs to substitute them with generated accessors.
- **`BusinessLayer/UserService.cs`** — the main caller. It instantiates `new User(firstName, lastName)` in `AddUser` and reads `Id`, `FirstName`, and `LastName` in `GetActiveUsers`.

### Behavior & Details

- **`User(string firstName, string lastName)`** is the constructor application code uses. It stores both names and sets `IsActive = true`, so every user created through this path starts active. There is no validation here — `UserService.AddUser` guards against null or whitespace names with `Guard` assertions before constructing the entity.
- **The `protected` parameterless constructor** exists because NHibernate materializes entities via reflection without calling your constructor. It assigns `FirstName = null!` and `LastName = null!` to satisfy nullable-reference-type analysis; the `null!` suppresses warnings without asserting a real value, since NHibernate populates the actual values right after construction. Marking it `protected` keeps application code from accidentally creating an unpopulated user.
- **`FirstName`, `LastName`, `IsActive`** are plain `virtual` auto-properties with no logic. Their `virtual` modifier is not for subclassing — it is a requirement of the NHibernate lazy-loading mechanism, which generates a subclass at runtime.
- The `Id` property comes from `Entity<int>`; it is `[Browsable(false)]` and its setter is `protected`, so application code can read it but never assign it — NHibernate fills it from the database identity value.

### Rough edges

- `UserService.AddUser` uses the public constructor, which always sets `IsActive = true`; there is currently no supported way to persist an inactive user or reactivate one, and `ShowUsersViewModel` treats `Updated`/`Deleted` as `NotImplementedException`.
- `GetActiveUsers` is named for active users but its LINQ query applies no filter on `IsActive`, so it returns every user.