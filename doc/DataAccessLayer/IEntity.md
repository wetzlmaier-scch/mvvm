# DataAccessLayer/IEntity.cs

### Purpose

This file defines `IEntity<TKey>`, the marker contract for anything the data access layer treats as an entity: a type that has an `Id` of a struct type. It exists so that the generic repository and unit-of-work types can constrain their `TEntity` parameters to "things with an identity" without knowing the concrete shape of each domain entity.

### Role in the Codebase

It is the base contract of the `DataAccessLayer` project, sitting next to the abstract base class `Entity<TKey>` (in `DataAccessLayer/Entity.cs`) that implements it. Every repository operation in this project is gated on this contract — for example `IRepository<TKey>.GetQueryable<TEntity>(...)` and `FindOne<TEntity>(...)` in `DataAccessLayer/IRepository.cs` all declare `where TEntity : class, IEntity<TKey>`, as does `HibernateRepository` in `DataAccessLayer.Hibernate/HibernateRepository.cs`. If this interface were removed, those constraints and `Entity<TKey>`'s implementation clause would stop compiling.

### Relationships

- Implemented by `DataAccessLayer/Entity.cs`'s `Entity<TKey>`, which is the abstract base class concrete domain entities are expected to derive from.
- Referenced as a generic constraint throughout `DataAccessLayer/IRepository.cs`, `IRepositoryFactory.cs`, `IUnitOfWork.cs`, and the Hibernate implementation, so the data-access layer can be written once against any entity type.
- Callers are any domain model that needs persistence: a concrete entity either derives from `Entity<TKey>` or implements this interface directly (which requires supplying its own `TKey Id { get; }`).

### Behavior & Details

- `TKey` is constrained to `struct`, so keys must be value types — `int`, `Guid`, `long`, etc. No reference-type keys.
- The interface is declared `out TKey` (covariant), allowing an `IEntity<int>` to be used where `IEntity<...>` is expected, and permitting `Entity<TKey>` to implement it with a property that is readable.
- `Id` has a getter only. A type implementing this interface directly must expose a read-only identity; it cannot set it through the interface. `Entity<TKey>` fills that gap by giving derived classes a `protected set`.
- Being a marker-style interface (a single property), it carries no behaviour of its own — its value is entirely as a constraint that unifies the repository layer around one idea of "entity".