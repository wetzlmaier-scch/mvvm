# `DataAccessLayer/IRepositoryFactory.cs`

### Purpose

This file defines `IRepositoryFactory<TRepository, TKey>`, the engine-agnostic contract for creating and dropping repository instances in the data access layer. It exists so that consumers (the business layer) can depend on "give me a repository" without knowing how sessions are opened — that detail belongs to a concrete persistence implementation, currently NHibernate.

### Role in the Codebase

The `DataAccessLayer` project is a small contracts-only project: it holds the persistence abstractions (`IRepository<TKey>`, `IUnitOfWork`, `IEntity<TKey>`) with no implementation. This interface is one of its core contracts — the factory side of the repository abstraction — and sits at the top of a three-level inheritance chain:

1. `IRepositoryFactory<TRepository, TKey>` (this file) — the generic, engine-agnostic contract.
2. `IHibernateRepositoryFactory<TKey>` in `DataAccessLayer.Hibernate/IHibernateRepositoryFactory.cs` — closes `TRepository` to `IHibernateRepository<TKey>`.
3. `IRepositoryFactory` in `BusinessLayer/IRepositoryFactory.cs` — closes `TKey` to `int`, the form business services actually consume.

Removing this interface would break both the Hibernate factory chain and the business-layer `IRepositoryFactory`, since they all ultimately derive from it.

### Relationships

- **Depends on** `IRepository<TKey>` (`DataAccessLayer/IRepository.cs`, same project): the `TRepository` parameter is constrained to `IRepository<TKey>`, so anything a factory can create must expose the query contract (`GetQueryable`, `Single`, `Find`, etc.). `TKey` is additionally constrained to `struct`.
- **Implemented by** `HibernateRepositoryFactory<TKey>` (`DataAccessLayer.Hibernate/HibernateRepositoryFactory.cs`), which satisfies the base `Create()` via an explicit interface implementation delegating to its public `Create()` method, and provides a `Drop()` implementation.
- **Consumed indirectly by** business-layer services. For example, `UserService` (`BusinessLayer/UserService.cs`) receives the non-generic `BusinessLayer.IRepositoryFactory` in its MEF `[ImportingConstructor]` and calls `Create()` per operation to open repositories for its queries.

### Behavior & Details

The interface has exactly two members:

- `TRepository Create()` — produce a repository instance. Because `TRepository` is declared as an `out` variance position, the interface only ever hands repositories *out*; consumers can never pass one back in. The concrete Hibernate implementation opens a new NHibernate stateless session per call, so each `Create()` yields an independent repository and callers are responsible for disposing it.
- `void Drop()` — intended to tear down / release factory-level resources (the session factory in the Hibernate implementation). In practice, `HibernateRepositoryFactory<TKey>.Drop()` throws `NotImplementedException`, so this member currently has no working implementation and should not be relied on for shutdown logic.

A consumer typically uses it like this: call `Create()`, use the returned repository inside a `using` block, and let disposal of the repository (not the factory) clean up the session.

### Rough edges

- `Drop()` is declared in the contract but the only implementation (`HibernateRepositoryFactory<TKey>.Drop()`) throws `NotImplementedException`, so the contract cannot actually be honored — code that calls `Drop()` expecting a graceful shutdown will crash.
- The factory is designed for a single instantiation: `HibernateRepositoryFactory<TKey>` sets a `static IsInitialized` flag and throws `InvalidOperationException` if a second instance is ever constructed, which is an implementation constraint the interface contract itself does not express.