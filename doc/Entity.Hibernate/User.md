# `Entity.Hibernate/User.cs`

### Purpose

This class is the NHibernate-annotated domain model for a user: it exists to be persisted to the `Users` table by NHibernate. It carries the name and an `IsActive` flag, and it always marks a newly created user as active. It is the storage-side counterpart of `UserDto` — the transport/DTO type used across the business and UI layers.

### Role in the Codebase

`User` is one of two files in the `Entity.Hibernate` project, whose job is to hold the NHibernate entity classes and their Fluent mappings. Its companion `UserMapping.cs` (a FluentNHibernate `ClassMap<User>`) declares the table name, identity-based `Id` generation, and column constraints (both name columns `not null`, length 50). Remove this class and the mapping, the repository, and the business layer all lose their user entity — the user-management feature is built entirely around it.

### Relationships

- **Base type:** `Entity<TKey>` (in `DataAccessLayer/Entity.cs`) — an abstract, `[Serializable]` base that contributes the `virtual Id` property with a protected setter, so the id is assigned only by the persistence layer (NHibernate, via `.GeneratedBy.Identity()` in `UserMapping`).
- **Caller:** `BusinessLayer/UserService.cs` is the only place that constructs `User`. In `AddUser(UserDto)` it runs `repository.Save(new User(user.FirstName, user.LastName))` inside a transaction; `GetActiveUsers()` reads `User` through the repository's queryable API. The ViewModel/test layers only ever see `UserDto`, never this class.

### Behavior & Details

Two constructors, with a deliberate split of responsibility:

- `protected User()` — required so NHibernate can materialize rows from the database. It sets `FirstName` and `LastName` to `null!` because NHibernate then overwrites them from the mapped columns. It does **not** touch `IsActive`; that property defaults to `false` at instantiation and is then overwritten from the column, so the in-code default never survives a load.
- `public User(string firstName, string lastName)` — the only way application code creates a user. It copies both names and sets `IsActive = true`, so every user persisted through `UserService.AddUser` starts active. There is no other code path that creates a `User`.

Notable behaviours and caveats:

- **All properties are `virtual`** — this is required so NHibernate can intercept reads/writes for lazy loading. Keeping them `virtual` is a hard constraint if you edit this class.
- **`Id` is read-only from this class.** The property is inherited from `Entity<int>` and has a `protected set` — the application code can read `u.Id` (e.g. when building a `UserDto`) but cannot assign it; identity generation is the database's job.
- **`IsActive` is write-only from application code in practice.** Nothing in the repository currently sets it to `false` — it is initialized to `true` at creation and only ever read back from the database. (Note also that `GetActiveUsers()` does not actually filter on `IsActive`; it selects all users despite its name.)
- **No validation on the entity itself.** Whitespace checks on the names happen in `UserService.AddUser` before a `User` is constructed, and database-level `not null` constraints are declared in `UserMapping`. The entity performs no checks of its own.

### Rough edges

- The protected constructor's `FirstName = null!; LastName = null!;` assignment is dead in practice — NHibernate overwrites both properties when it hydrates an entity, so the assignments serve no purpose.
- `GetActiveUsers()` in `UserService` never filters by `IsActive`, so the property is effectively write-on-creation with no consumer — worth checking whether that is intended.