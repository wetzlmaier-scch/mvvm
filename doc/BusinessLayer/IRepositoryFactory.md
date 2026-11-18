# `BusinessLayer/IRepositoryFactory.cs`

### Purpose

This file defines `IRepositoryFactory`, the business layer's own, non-generic contract for creating NHibernate repositories. It exists so that business services can depend on a factory type with the key type already closed to `int` — the application's actual entity key type — instead of carrying the generic parameter of the data-access layer through every business method.

It is an empty marker interface: it adds no members of its own. Its entire job is inheritance, which pulls the generic contract in and pins it down.

### Role in the Codebase

It is a small contract file inside `BusinessLayer/`, alongside `RepositoryFactory` (the MEF-exported implementation) and the business services that consume it. It forms the seam between the business layer and the NHibernate-based data access: services program against this interface, never against the `DataAccessLayer` or `DataAccessLayer.Hibernate` types directly.

If it were removed, `UserService` and `RepositoryFactory` would have no business-layer type to compile against, and services would be forced to depend on the generic `IHibernateRepositoryFactory<int>` from `DataAccessLayer.Hibernate` instead.

### Relationships

- **Extends** `IHibernateRepositoryFactory<TKey>` (defined in `DataAccessLayer.Hibernate/IHibernateRepositoryFactory.cs`), with `TKey` closed to `int`.
- Through that base it inherits from `DataAccessLayer.IRepositoryFactory<TRepository, TKey>` (defined in `DataAccessLayer/IRepositoryFactory.cs`).
- The members a consumer of this interface sees are therefore:
  - `IHibernateRepository<int> Create()` — declared by `IHibernateRepositoryFactory<TKey>`, which hides the base `Create()`; calling it through this interface returns the NHibernate-specific repository, which extends `IRepository<TKey>` and adds `Save<TEntity>(TEntity)` and `UnitOfWork`.
  - `void Drop()` — from the base `DataAccessLayer.IRepositoryFactory<TRepository, TKey>`.
- **Implemented by** `RepositoryFactory` (`BusinessLayer/RepositoryFactory.cs`), which extends `HibernateRepositoryFactory<int>` and is MEF-exported as `[Export(typeof(IRepositoryFactory))]`.
- **Consumed by** `UserService` (`BusinessLayer/UserService.cs`), which takes `IRepositoryFactory` in its `[ImportingConstructor]` and calls `_repositoryFactory.Create()` inside a `using` block for each operation.

### Behavior & Details

Because the interface is empty, its behaviour is entirely that of its generic base chain. Two things are worth knowing for a caller:

- Calling `Create()` through this interface yields `IHibernateRepository<int>`, so callers get the NHibernate-specific members (`Save`, `UnitOfWork`) for free without referring to the `DataAccessLayer.Hibernate` namespace.
- `IHibernateRepository<TKey>` extends `IDisposable`, which is why `UserService` wraps every `Create()` call in a `using` block.
- `TKey` is constrained to `struct` up the chain, which is why `int` (and any other struct) is a valid closing; the application standardises on `int`.
- `Drop()` is inherited from `DataAccessLayer.IRepositoryFactory<TRepository, TKey>`. No caller in the business layer uses it, and the concrete `HibernateRepositoryFactory` provides no working shutdown logic for it, so treat it as effectively vestigial in this layer.

### Rough edges

- The name `IRepositoryFactory` shadows `DataAccessLayer.IRepositoryFactory<TRepository, TKey>` in files that import both namespaces; code in this project resolves it by namespace qualification or by not importing `DataAccessLayer` at all.
- `Drop()` exists in the contract but has no working implementation in the concrete layers, so it cannot be relied on for shutdown logic.