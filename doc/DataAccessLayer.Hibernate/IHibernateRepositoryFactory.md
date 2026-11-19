# `DataAccessLayer.Hibernate/IHibernateRepositoryFactory.cs`

### Purpose

This file defines `IHibernateRepositoryFactory<TKey>`, the contract for a factory that produces NHibernate-backed repositories. It is a thin specialization of the generic `IRepositoryFactory<TRepository, TKey>`: it closes `TRepository` to `IHibernateRepository<TKey>` and re-declares `Create()` so callers of this interface get the NHibernate-specific repository type directly, rather than the generic base's repository type.

### Role in the Codebase

It sits in the `DataAccessLayer.Hibernate` module, which is the NHibernate implementation layer of the data access stack. `HibernateRepositoryFactory<TKey>` in the same folder is its concrete implementation, and the interface exists so upper layers can depend on "a factory of NHibernate repositories" without referencing the concrete class.

The interface is the middle rung of a small chain:

1. `IRepositoryFactory<TRepository, TKey>` in `DataAccessLayer/IRepositoryFactory.cs` — fully generic.
2. `IHibernateRepositoryFactory<TKey>` here — closes `TRepository` to `IHibernateRepository<TKey>`.
3. `BusinessLayer/IRepositoryFactory.cs` — a non-generic marker interface that extends `IHibernateRepositoryFactory<int>` so business-layer code (e.g. `UserService`) can depend on a repository factory for `int`-keyed entities without knowing about NHibernate or generics.

### Relationships

- **Extends** `IRepositoryFactory<TRepository, TKey>` (defined in `DataAccessLayer/IRepositoryFactory.cs`), with `TRepository` fixed to `IHibernateRepository<TKey>`. From the base it inherits `Drop()` and the `Create()` member it hides.
- **References** `IHibernateRepository<TKey>` (same folder), the NHibernate-specific repository interface with `Save<T>` and `UnitOfWork`.
- **Implemented by** `HibernateRepositoryFactory<TKey>` (same folder).
- **Consumed by** `BusinessLayer/IRepositoryFactory.cs`, which derives from `IHibernateRepositoryFactory<int>`.

### Behavior & Details

- `new IHibernateRepository<TKey> Create();` — the `new` keyword hides the base `IRepositoryFactory<TRepository, TKey>.Create()` rather than overriding it (interfaces cannot override). A caller holding the `IHibernateRepositoryFactory` interface gets the NHibernate repository type; a caller holding only the base interface sees the generic `Create()`.
- The `where TKey : struct` constraint is repeated from the base, which is what lets `TKey` be `int` in the business-layer specialization.

### Rough edges

The `new`-hidden `Create()` is redundant in practice: the base constraint `where TRepository : IRepository<TKey>` plus the fixed `TRepository` argument already means a conforming type's generic `Create()` returns `IHibernateRepository<TKey>`. The re-declaration mostly exists for interface-level clarity. It does create an overload trap: code that holds the base `IRepositoryFactory<IHibernateRepository<TKey>, TKey>` reference calls the generic `Create()`, while code holding the derived interface calls the hidden one — the concrete `HibernateRepositoryFactory<TKey>` implements both bodies (the second via explicit interface implementation, delegating to the first), which is easy to miss when extending the factory.