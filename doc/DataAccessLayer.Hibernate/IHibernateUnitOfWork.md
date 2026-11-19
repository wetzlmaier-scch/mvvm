# `DataAccessLayer.Hibernate/IHibernateUnitOfWork.cs`

### Purpose

This interface defines the NHibernate-specific unit-of-work contract: a provider-specific extension of the base `IUnitOfWork` (`DataAccessLayer/IUnitOfWork.cs`) that lets callers start a transaction with an explicit `System.Data.IsolationLevel`. The base `IUnitOfWork` only offers a parameterless `BeginTransaction()`, so any code that needs to control isolation (for example, switching between `ReadCommitted` and `Serializable`) must program against this narrower interface instead.

The interface adds a single member on top of `IUnitOfWork`:

- `void BeginTransaction(IsolationLevel isolationLevel)` — begin a transaction at the given isolation level.

Everything else a unit of work must provide — `IsInTransaction`, the parameterless `BeginTransaction()`, `RollBackTransaction()`, `CommitTransaction()`, and `Dispose()` (via `IUnitOfWork`'s `IDisposable` inheritance) — is inherited unchanged from `IUnitOfWork`.

### Role in the Codebase

This is one of the seam interfaces in the `DataAccessLayer.Hibernate` folder, alongside `IHibernateRepository`, `IHibernateRepositoryFactory`, and `IModelContext`. Its job is to let the rest of the application depend on transaction behaviour without naming NHibernate types directly: `IStatelessSession` and `ITransaction` never appear in the interface. The concrete implementation is `HibernateUnitOfWork` (`DataAccessLayer.Hibernate/HibernateUnitOfWork.cs`), which is the only class in this codebase that implements the interface.

If this interface were removed, the only thing lost would be the isolation-level overload of `BeginTransaction`; the repository would fall back to exposing the base `IUnitOfWork`, and callers needing a specific isolation level would have no way to request one through the abstraction.

### Relationships

- **Extends** `IUnitOfWork` (`DataAccessLayer/IUnitOfWork.cs`) in the provider-neutral base project, so implementers must satisfy both contracts.
- **Implemented by** `HibernateUnitOfWork` (`DataAccessLayer.Hibernate/HibernateUnitOfWork.cs`), which delegates the call to `IStatelessSession.BeginTransaction(isolationLevel)` and defaults the parameterless overload to `IsolationLevel.ReadCommitted`.
- **Exposed through** `IHibernateRepository<TKey>.UnitOfWork` (`DataAccessLayer.Hibernate/IHibernateRepository.cs`), whose implementation in `HibernateRepository<TKey>` lazily creates a `HibernateUnitOfWork` sharing the repository's `IStatelessSession`. That is the main path by which repository consumers reach this contract: they take a repository, access its `UnitOfWork` property, and can call the isolation-level overload through it.

### Behavior & Details

The interface itself has no logic, but its contract is worth knowing before implementing or relying on it:

- An implementer must also honour `IDisposable` (inherited through `IUnitOfWork`), since a unit of work owns the NHibernate `ITransaction` it started.
- Because it adds a method rather than replacing `BeginTransaction()`, both overloads exist side by side; callers on the base `IUnitOfWork` get the default behaviour, callers on `IHibernateUnitOfWork` can pick the level.
- In `HibernateUnitOfWork`, `BeginTransaction(IsolationLevel)` throws `InvalidOperationException` if a transaction is already running, so the isolation-level overload cannot be used to "up-grade" or nest an open transaction — it is a fresh start only.

### Rough edges

None of the interfaces in this folder carry XML doc comments, and the isolation-level overload here has no documentation of its own, so the reason the extension exists is only discoverable by comparing it with the base `IUnitOfWork`.