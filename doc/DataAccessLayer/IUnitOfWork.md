# `DataAccessLayer/IUnitOfWork.cs`

### Purpose

`IUnitOfWork` is the provider-agnostic contract for database transactions. It lets a caller open, commit, and discard a transaction without knowing which persistence engine is doing the work, so a group of data operations can be treated as one atomic unit. It has no implementation of its own — it sits in the `DataAccessLayer` contracts project, which holds persistence abstractions with no logic.

### Role in the Codebase

This is the transaction half of the `DataAccessLayer` contract set, alongside the read-oriented `IRepository<TKey>` and the entity types (`IEntity<TKey>`, `Entity<TKey>`). The contract set deliberately keeps the persistence engine out of the signatures that higher layers depend on, and `IUnitOfWork` is how those higher layers get transaction control without that coupling.

If it were removed, the NHibernate layer would lose its base contract: `IHibernateUnitOfWork` in `DataAccessLayer.Hibernate/IHibernateUnitOfWork.cs` extends it directly, and `HibernateUnitOfWork` implements all of its members.

### Relationships

- **`System.IDisposable`** (standard .NET): `IUnitOfWork` extends it, so a unit of work is expected to release its underlying session/transaction resources when disposed.
- **`IHibernateUnitOfWork`** (`DataAccessLayer.Hibernate/IHibernateUnitOfWork.cs`): extends this interface and adds one member, `BeginTransaction(IsolationLevel)`, for choosing an isolation level beyond the default.
- **`HibernateUnitOfWork`** (`DataAccessLayer.Hibernate/HibernateUnitOfWork.cs`): the concrete class implementing every member here, wrapping an NHibernate `IStatelessSession`.
- **`IHibernateRepository<TKey>`** (`DataAccessLayer.Hibernate/IHibernateRepository.cs`): exposes the unit of work through its `UnitOfWork` property. `HibernateRepository<TKey>` lazily creates a `HibernateUnitOfWork` for that property and disposes it in its own `Dispose()`.
- **Business callers**: `BusinessLayer/UserService.AddUser` is the in-repo example. It gets a repository, calls `repository.UnitOfWork.BeginTransaction()`, saves within it, and commits on success; on any exception it rolls back, logs, and rethrows a `BusinessLayerException`. (It calls through the `IHibernateUnitOfWork` type, which carries these same members via `IUnitOfWork`.)

### Behavior & Details

The contract has four members, and they define a simple state machine:

- **`bool IsInTransaction { get; }`** — reports whether a transaction is currently open. The implementation derives this from whether a transaction has been started and not yet released.
- **`BeginTransaction()`** — opens a transaction. The implementation (`HibernateUnitOfWork`) starts with `ReadCommitted` isolation and throws `InvalidOperationException` if a transaction is already running, so at most one transaction is active at a time.
- **`CommitTransaction()`** — makes the open transaction permanent. The implementation throws `InvalidOperationException` if no transaction is running, and rolls back before rethrowing if the commit itself fails.
- **`RollBackTransaction()`** — discards the open transaction; also throws `InvalidOperationException` when there is nothing to roll back.

Because the interface is provider-neutral, it says nothing about isolation levels — that is where the thinner `IHibernateUnitOfWork` extension comes in. Program against `IUnitOfWork` for default behaviour and against `IHibernateUnitOfWork` when you need to choose an isolation level. The usage pattern visible in the callers is: begin, do the work, commit; on any exception, roll back and let the caller handle the error.

### Rough edges

- The interface offers no way to express an isolation level, so a provider that needs it has to introduce its own `BeginTransaction(IsolationLevel)` overload rather than extending the shared contract.
- The base contract is defined and extended here, but the only in-repo consumer (`UserService.AddUser`) accesses the members through the derived `IHibernateUnitOfWork` type; no code holds `IUnitOfWork` directly.