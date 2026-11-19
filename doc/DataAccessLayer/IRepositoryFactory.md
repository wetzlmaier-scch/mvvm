# `DataAccessLayer/IRepositoryFactory.cs`

### Purpose

This file defines `IRepositoryFactory<TRepository, TKey>`, the engine-agnostic contract for producing repository instances. It exists so that higher layers can depend on "give me a repository, let me tear it down" without naming NHibernate, `ISessionFactory`, or any other persistence mechanism. It is the base link in a three-level factory interface chain: this generic interface, `IHibernateRepositoryFactory<TKey>` (`DataAccessLayer.Hibernate/IHibernateRepositoryFactory.cs`) which closes the repository type to `IHibernateRepository<TKey>`, and the empty non-generic `IRepositoryFactory` in `BusinessLayer/IRepositoryFactory.cs` which closes `TKey` to `int`.

### Role in the Codebase

`DataAccessLayer` is a small contract-only project: interfaces and marker types (`IRepository<TKey>`, `IEntity<TKey>`, `Entity<TKey>`, `IUnitOfWork`, `IRepositoryFactory`) with no behavior. This interface is the creation half of that contract set — `IRepository<TKey>` defines what a repository *does*, and this interface defines how one is *obtained and disposed*. If it were removed, `IHibernateRepositoryFactory<TKey>` and therefore the business layer's `IRepositoryFactory` and `RepositoryFactory` would all stop compiling.

### Relationships

- **Depends on** `IRepository<TKey>` (same project, `DataAccessLayer/IRepository.cs`): the constraint `TRepository : IRepository<TKey>` ties any repository this factory hands out to the data-layer query contract.
- **Extended by** `IHibernateRepositoryFactory<TKey>` in `DataAccessLayer.Hibernate`, which inherits this interface as `IRepositoryFactory<IHibernateRepository<TKey>, TKey>` and adds a `new` (hiding) `Create()`.
- **Implemented, transitively, by** `HibernateRepositoryFactory<TKey>` (`DataAccessLayer.Hibernate/HibernateRepositoryFactory.cs`), which satisfies this interface's `Create()` via an explicit interface implementation that delegates to its public `Create()`.
- **Consumed indirectly by** business services. `UserService` (`BusinessLayer/UserService.cs`) receives the closed non-generic `IRepositoryFactory` through its MEF constructor and calls `Create()` for each operation.

### Behavior & Details

The interface declares two members:

- **`TRepository Create()`** — return a ready-to-use repository. Callers hold the result (typically in a `using` block, as `UserService` does) and dispose it when done; the factory is not a cache, so each call yields a fresh instance. The covariance on `TRepository` (`out TRepository`) lets a factory closed to a more specific repository type satisfy a consumer expecting a broader one.
- **`void Drop()`** — release the resources the factory itself holds (in the concrete `HibernateRepositoryFactory`, that would be the `ISessionFactory`). Note this is distinct from disposing a repository obtained from `Create()`.

Constraints: `TKey : struct` mirrors the constraint on `IRepository<TKey>` so factory and repository stay mutually compatible.

### Rough edges

- `Drop()` is part of the contract but the only concrete implementation (`HibernateRepositoryFactory.Drop()`) throws `NotImplementedException`, so any caller invoking it on the resolved factory will fail at runtime. There is no working session-factory teardown path in the repository.
- The name is shadowed one level up: `BusinessLayer.IRepositoryFactory` is a different, non-generic type with the same simple name, so code importing both `DataAccessLayer` and `BusinessLayer` must disambiguate by namespace.