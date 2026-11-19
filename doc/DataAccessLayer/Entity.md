# `DataAccessLayer/Entity.cs`

### Purpose

This file defines `Entity<TKey>`, the abstract base class that gives every persisted entity in the system its identity. It exists so that all domain entities share one uniform, read-mostly `Id` property with a value-type key, rather than each entity declaring its own.

### Role in the Codebase

`DataAccessLayer` is a contracts-only project: persistence interfaces (`IRepository.cs`, `IRepositoryFactory.cs`, `IUnitOfWork.cs`), the marker interface `IEntity<TKey>` (`IEntity.cs`), a shared `DataAccessException`, and this base class. `Entity<TKey>` is the concrete half of the entity contract — `IEntity<TKey>` declares what an entity is, `Entity<TKey>` provides the one member the interface requires so concrete models don't have to repeat it.

Concrete entities in the entity projects derive from it, each picking its own key type — e.g. `User : Entity<int>` in `Entity.Hibernate/User.cs`. If this class were removed, every entity would have to declare its own `Id` property (or implement `IEntity<TKey>` directly) to remain usable by the repositories.

### Relationships

- **Implements `IEntity<TKey>`** (same project, `IEntity.cs`) — declares only `TKey Id { get; }` and is marked `out` (covariant). The covariance is what lets this class satisfy the interface with a `virtual` property that has a `protected set`: the setter is simply hidden from the interface view, not part of the contract.
- **Subclasses in the entity projects** — `User : Entity<int>` in `Entity.Hibernate/User.cs` is the pattern; the key type is chosen per entity.
- **`DataAccessLayer.Hibernate/HibernateRepository.cs`** — the NHibernate implementation constrains every generic method to `TEntity : class, IEntity<TKey>` (the interface, not this class). So any type exposing an `Id` of the right type can flow through the repository; deriving from `Entity<TKey>` is the convenient way to do it, not a strict requirement.

### Behavior & Details

- `Entity<TKey>` is abstract and constrains `where TKey : struct`, so key types are value types (`int`, `Guid`, ...) — mirroring the constraint on `IEntity<TKey>` and on the repositories.
- The parameterless `protected` constructor sets `Id = default!` (the `!` suppresses the nullable warning rather than setting a meaningful value — `default(int)` is `0`, `default(Guid)` is empty). Derived entities must provide their own constructors, so `Id` is at its key-type default until the database assigns it — the NHibernate mappings use identity generation (e.g. `Id(x => x.Id).GeneratedBy.Identity()`), which fills `Id` on insert.
- **`Id`** is `virtual` with a `protected set` and `[Browsable(false)]`. The protected setter means application code and repository code can only read it; only the entity class hierarchy (or an ORM setting the value via reflection) can write it. The `[Browsable(false)]` hides it from `TypeDescriptor`-based tooling such as designers and object viewers.
- The `[Serializable]` attribute marks the class for .NET serialization. `IEntity<TKey>` itself carries no such attribute, so only entities deriving from `Entity<TKey>` (not every `IEntity<TKey>` implementer) get it.

### Rough edges

The `virtual` on `Id` permits a derived class to override the property with a public setter, which compiles even though the interface and this base class only promise read-only identity. Nothing in the type system enforces "identity is assigned by the database, not by application code" — the `protected set` here is a convention a subclass can silently weaken.