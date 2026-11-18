### Purpose

`DataAccessLayer/IEntity.cs` defines `IEntity<TKey>`, the marker contract for anything the data access layer treats as a persistable entity: a reference type exposing a read-only `Id` whose type is a struct. It exists so the generic repository types can be written once against a single constraint — `TEntity : class, IEntity<TKey>` — instead of naming concrete entity classes.

### Role in the Codebase

It is the foundational contract of the `DataAccessLayer` project, which is otherwise a small set of persistence abstractions (`IRepository`, `IRepositoryFactory`, `IUnitOfWork`, `Entity<TKey>`). The abstract base class `Entity<TKey>` in `DataAccessLayer/Entity.cs` implements this interface, and concrete domain models (for example `User` in `Entity.Hibernate`) inherit from `Entity<TKey>`. Every method in `IRepository<TKey>` (`DataAccessLayer/IRepository.cs`) and `IHibernateRepository` constrains `TEntity` to `class, IEntity<TKey>`, so a type that does not implement this interface simply cannot be passed to the repository. Removing it would break compilation across the data access layer, its Hibernate implementation, and the domain models.

### Relationships

- **Implementer:** `Entity<TKey>` (`DataAccessLayer/Entity.cs`) — the abstract base class that provides the `Id` property, so concrete models like `User` (in `Entity.Hibernate`) get the implementation for free by extending it.
- **Consumers:** `IRepository<TKey>` (`DataAccessLayer/IRepository.cs`) — all seven of its query methods constrain `TEntity : class, IEntity<TKey>`. The NHibernate implementation `DataAccessLayer.Hibernate/HibernateRepository.cs` repeats the same constraint on its `Get`, `GetQueryable`, `Find`, `Save`, and similar methods. The constraint is applied to the type, not the interface, so nothing here is inherited — `IEntity<TKey>` is purely a compile-time gate.
- **No imports:** the interface itself depends only on `System` (for the `struct` constraint). It pulls in no other code.

### Behavior & Details

The interface has exactly one member: `TKey Id { get; }`, constrained to `TKey : struct` so keys must be value types (e.g. `int`, `Guid`), never `null`.

Two declarations do real work:

- **Read-only `Id`.** There is no setter, so code outside the entity class cannot reassign identity — it is assigned by the data layer on save or populated from the database on load, not by business code.
- **`out TKey` variance.** The type parameter is covariant, which permits `Entity<TKey>` to implement the interface even though its `Id` is `virtual` with a `protected set` (the setter is hidden from the public surface and is not part of the interface contract). In practice `Id` flows out of the entity (read for comparison, persistence bookkeeping) and never back in.

A type is an entity under this contract if and only if it exposes a public, get-only `TKey Id` with a struct key. That is the entire thing the persistence layer requires of it.

### Rough edges

`IEntity<TKey>` declares only a getter, but the `virtual` `Id` on `Entity<TKey>` could still be overridden in a derived class with a public setter — that would compile even though the interface never promised one, quietly weakening the "identity is read-only" expectation the interface advertises.