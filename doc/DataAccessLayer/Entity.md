# `DataAccessLayer/Entity.cs`

### Purpose

This file defines `Entity<TKey>`, the abstract base class that gives every persisted entity in the system its identity. It exists so that all domain entities share a single, uniform `Id` property with a consistent contract, without each entity having to declare its own key.

### Role in the Codebase

`DataAccessLayer` is a small contract-and-abstraction project: interfaces for repositories (`IRepository.cs`, `IRepositoryFactory.cs`, `IUnitOfWork.cs`), the entity marker interface `IEntity<TKey>` (`IEntity.cs`), plus `DataAccessException` and `SortOrder`. This class is the concrete half of the entity-identity contract: `IEntity<TKey>` declares the interface, and `Entity<TKey>` is the base class real entities inherit from. If it were removed, concrete entities (e.g. `Entity.Hibernate/User.cs`, `User : Entity<int>`) would lose their shared `Id` implementation and would have to define keys individually.

### Relationships

- **`IEntity<TKey>`** (defined in `DataAccessLayer/IEntity.cs`) — the interface this class implements. It declares only `TKey Id { get; }` and has an `out` variance modifier, so `IEntity<int>` values flow into read-only collections and repository queries.
- **Subclasses in the entity projects** — e.g. `User : Entity<int>` in `Entity.Hibernate/User.cs` is the pattern: each concrete entity picks a struct key type and inherits `Id`.
- **`DataAccessLayer.Hibernate/HibernateRepository.cs`** — the repository methods are constrained to `IEntity<TKey>` (the interface, not this class), so anything exposing an `Id` of the right key type works with the data-access layer, but this class is the intended way for entities to satisfy that constraint.

### Behavior & Details

- `Entity<TKey>` is abstract, so it can only be subclassed. The `where TKey : struct` constraint restricts key types to value types (e.g. `int`, `Guid`).
- The constructor is `protected` and sets `Id = default!` (the C# null-forgiving operator applied to `default(TKey)`, e.g. `0` for `int`). Subclasses must be constructible without arguments (see `User`'s comment about NHibernate requiring a parameterless constructor), and that requirement chains from here.
- `Id` is `public virtual` with a `protected set` — meaning only the class itself or derived classes can assign it, never application code or the repositories from outside the hierarchy. Declaring it `virtual` leaves room for a persistence provider or subclass to override the accessor if needed, though the known subclasses (`User`) do not.
- `[Serializable]` on the class marks it for binary/serialization compatibility, and `[Browsable(false)]` on `Id` keeps the key property out of designer/property-grid tooling.
- Nothing is stored or cached: there is no state beyond the key, no side effects, and no failure modes of its own. A new `User` therefore starts with `Id == 0` until the provider assigns a real key.

### Rough edges

`IEntity<TKey>`'s `Id` setter is intentionally absent, yet `Entity<TKey>` marks `Id` `virtual` — a derived class could in principle override the property with a public setter, breaking the "identity is not settable from outside" contract. Nothing in this codebase does so, but the compiler will not stop a subclass from doing it.