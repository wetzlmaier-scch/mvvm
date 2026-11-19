# Entity.Hibernate/User.cs

### Purpose
This file defines the `User` class, the NHibernate entity that represents a user in the persistence layer. It exists so that `UserService` can persist and load users against the database, while the rest of the application works with separate DTOs and view models rather than the entity itself.

### Role in the Codebase
`User` is the only entity in the `Entity.Hibernate` project (alongside its NHibernate mapping, `UserMapping.cs`, which maps it to the `Users` table). It sits at the bottom of the layering: `BusinessLayer/UserService.cs` is the only code that constructs it — `AddUser` runs `repository.Save(new User(user.FirstName, user.LastName))` inside a transaction, and the query methods project it into `UserDto`. UI view models never touch `User` directly. Removing it would break the business layer's persistence and the FluentNHibernate mapping that references it.

### Relationships
- **Base class:** `DataAccessLayer/Entity.cs` — `Entity<TKey>` is an abstract, `[Serializable]` base class in the same repository (namespace `DataAccessLayer`, pulled in by the `using DataAccessLayer` directive). It contributes the `virtual TKey Id` property (with a `protected` setter) and a constructor that initialises it. `User` derives from `Entity<int>`, so its `Id` is an `int` assigned by the database.
- **Mapping:** `Entity.Hibernate/UserMapping.cs` — a FluentNHibernate `ClassMap<User>` that maps `Id` to an identity-generated key and `FirstName`, `LastName`, `IsActive` to non-nullable columns on the `Users` table.
- **Caller:** `BusinessLayer/UserService.cs` is the sole constructor of `User` in this repository. `IEntity<TKey>` (referenced by the base class) was not located in the repository and is assumed to come with the `DataAccessLayer` dependency.

### Behavior & Details
Two constructors with a deliberate split:

- `protected User()` — exists only so NHibernate can instantiate the object when reading rows. It nulls out `FirstName` and `LastName` (`null!`) so the class passes nullable-reference-type analysis. Because the parameterless constructor is `protected` and the properties are plain `virtual` setters, the ORM sets the values after construction.
- `public User(string firstName, string lastName)` — the application-side constructor. It sets both names and defaults `IsActive = true`, so every user created through the business layer starts out active.

All properties are `virtual`, which NHibernate requires to intercept them via proxying and lazy loading; do not remove the `virtual` modifier. `Id` is not settable from application code — its setter is `protected` on the base class and the mapping uses `GeneratedBy.Identity()`, so the database assigns it on insert.

### Rough edges
`FirstName` and `LastName` are declared as nullable-reference `string`s, and the protected constructor assigns `null!` to satisfy the compiler. In practice NHibernate will always populate them (the mapping marks both columns non-nullable), but the type system allows null and nothing at runtime enforces the invariant.