# BusinessLayer/IRepositoryFactory.cs

### Purpose

This file defines `IRepositoryFactory`, the BusinessLayer's own, key-free contract for creating NHibernate repositories. It exists so that business services (like `UserService`) can depend on a factory abstraction without knowing that repositories are backed by NHibernate/Hibernate, or that the repository key type is `int`.

### Role in the Codebase

It is a small type alias / specialization interface inside the `BusinessLayer` namespace. It derives from `IHibernateRepositoryFactory<int>` (defined in `DataAccessLayer.Hibernate/IHibernateRepositoryFactory.cs`), which itself derives from the generic `DataAccessLayer.IRepositoryFactory<IHibernateRepository<TKey>, TKey>` and adds `Create()` returning `IHibernateRepository<TKey>`.

By fixing `TKey` to `int` and dropping the generic parameter, this interface gives the business layer a stable, non-generic seam for repository creation. If it were removed, `UserService` and `RepositoryFactory` would need to reference the generic `IHibernateRepositoryFactory<int>` directly, leaking the key type and the DataAccessLayer's Hibernate-specific naming into the business layer.

### Relationships

- **Extends** `IHibernateRepositoryFactory<int>` (from `DataAccessLayer.Hibernate`), which inherits from `DataAccessLayer.IRepositoryFactory<IHibernateRepository<TKey>, TKey>`. So members inherited through the chain include:
  - `IHibernateRepository<int> Create()` — from `IHibernateRepositoryFactory<TKey>`.
  - `void Drop()` — from the generic base `IRepositoryFactory<TRepository, TKey>`.
- **Implemented by** `RepositoryFactory` (same folder, `BusinessLayer/RepositoryFactory.cs`), which also extends `HibernateRepositoryFactory<int>` and is MEF-exported as `IRepositoryFactory`.
- **Consumed by** `UserService` (`BusinessLayer/UserService.cs`), which takes `IRepositoryFactory` in its `[ImportingConstructor]` and calls `_repositoryFactory.Create()` to open repositories for queries and unit-of-work transactions.

### Behavior & Details

The interface declares no new members of its own; it contributes only the type-level specialization (fixing `TKey` to `int`) and a non-generic name that business-layer code can reference. All contract behavior comes from the inherited members:

- `Create()` returns a new `IHibernateRepository<int>` (in practice a `HibernateRepository<int>` over a stateless session, per `HibernateRepositoryFactory.Create`). Callers treat it as disposable (`using var repository = _repositoryFactory.Create();`).
- `Drop()` is declared in the generic base but is never implemented meaningfully — `HibernateRepositoryFactory.Drop()` throws `NotImplementedException`, so callers in the business layer don't use it.
- Because `TKey` is constrained to `struct` in the base interfaces, `int` satisfies the constraint; that's why the specialization uses `int` (matching the integer primary keys of the entities, e.g. `User`).

One practical caveat for anyone implementing this: the underlying `HibernateRepositoryFactory` constructor enforces a process-wide single initialization (a static `IsInitialized` flag), so only one factory/session factory can be created per process.