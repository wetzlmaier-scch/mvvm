# `DataAccessLayer/Entity.cs`

### Purpose

This file defines `Entity<TKey>`, the abstract base class that gives every persisted entity in the system its identity. It exists so that all domain entities share a single, uniform `Id` property with a well-defined rule: identity is read-only to application code and is assigned by the persistence provider, never by the application.

### Role in the Codebase

`DataAccessLayer` is a small contract-and-abstraction project: interfaces for repositories (`IRepository.cs`, `IRepositoryFactory.cs`, `IUnitOfWork.cs`), the entity marker interface `IEntity<TKey>` (`IEntity.cs`), the exception type, and this base class. `Entity<TKey>` is the one concrete piece of behaviour in the project — the rest are contracts.

Concrete entities in separate projects derive from it, picking their key type: for example `User : Entity<int>` in `Entity.Hibernate/User.cs`. If this class were removed, every entity would have to re-declare an `Id` property and re-implement `IEntity<TKey>` by hand, and the "identity is assigned by the provider, not the app" guarantee would become a convention nobody enforces.

### Relationships

- **`IEntity<TKey>`** (defined in `DataAccessLayer/IEntity.cs`) — the interface this class implements. It declares only `TKey Id { get; }` and is marked `out` (covariant), which is exactly why `Entity<TKey>` can satisfy it: the implementation's `Id` property is read through the interface, and the `protected` setter never conflicts with the getter-only contract.
- **Subclasses in the entity projects** — `User : Entity<int>` in `Entity.Hibernate/User.cs` is the pattern: each concrete entity picks a struct key type and inherits `Id` for free.
- **`DataAccessLayer.Hibernate/HibernateRepository.cs`** — the repository's generic methods are constrained to `IEntity<TKey>` (the interface, not this class), so anything exposing an `Id` of the right key type flows through the repository, whether or not it derives from `Entity<TKey>`.

### Behavior & Details

- `Entity<TKey>` is abstract, so it can only be subclassed. The `where TKey : struct` constraint restricts key types to value types (e.g. `int`, `Guid`), mirroring the constraint on `IEntity<TKey>` and `IRepository<TKey>`.
- The parameterless constructor runs `Id = default!;`, i.e. sets `Id` to the key type's zero value (`0` for `int`, `Guid.Empty` for `Guid`). This is a defensive default so the property is never `null`; the persistence provider (NHibernate, per `Entity.Hibernate/UserMapping.cs` using `GeneratedBy.Identity()`) overwrites it from the database on load.
- `Id` is `virtual` with a `protected` setter: derived entity classes can read it and the provider/hierarchy can set it, but application code outside the class hierarchy cannot assign it. The `[Browsable(false)]` attribute hides the property from data-bound UI controls and designers — it is a hint for tooling, not a runtime behaviour change.
- The `[Serializable]` attribute marks the class for .NET serialization. Note that `IEntity<TKey>` carries no such attribute, so only entities deriving from `Entity<TKey>` get it.

### Rough edges

`IEntity<TKey>`'s `Id` setter is intentionally absent, yet `Entity<TKey>` marks `Id` `virtual` — a derived class could in principle override the property with a public setter, breaking the "identity is assigned by the provider" guarantee the class exists to enforce. Nothing in the code prevents that override.