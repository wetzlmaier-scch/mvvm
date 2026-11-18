### Purpose

`DataAccessLayer.Hibernate/IHibernateUnitOfWork.cs` defines `IHibernateUnitOfWork`, the NHibernate-specific unit-of-work contract. It exists because the generic `IUnitOfWork` only allows starting a transaction without choosing its isolation level, and NHibernate's `IBeginTransaction`/`BeginTransaction(IsolationLevel)` supports picking one. This interface adds exactly that capability: `BeginTransaction(IsolationLevel isolationLevel)`, letting callers control how strongly they isolate their reads and writes within a single unit of work.

### Role in the Codebase

Inside `DataAccessLayer.Hibernate/`, this file is the abstraction layer for transaction management alongside `IHibernateRepository` and `IHibernateRepositoryFactory`. Concrete class `HibernateUnitOfWork` (same directory) is the sole implementation — it wraps an `IStatelessSession` and stores the underlying NHibernate `ITransaction`. Business or service layers depend on `IHibernateUnitOfWork` rather than the concrete type, so they are decoupled from NHibernate specifics. If this interface were removed, `HibernateRepository.UnitOfWork` would need to expose `HibernateUnitOfWork` directly, leaking NHibernate's implementation details up to callers.

### Relationships

- **Extends** `IUnitOfWork` from `DataAccessLayer/IUnitOfWork.cs`, which itself extends `System.IDisposable` and declares `IsInTransaction`, `BeginTransaction()`, `RollBackTransaction()`, and `CommitTransaction()`. This file contributes exactly one additional member: `BeginTransaction(IsolationLevel isolationLevel)`.
- **Implemented by** `HibernateUnitOfWork` (`DataAccessLayer.Hibernate/HibernateUnitOfWork.cs`). The implementation uses `System.Data.IsolationLevel` from the `System.Data` namespace, matching the signature declared here.
- **Consumed via** `IHibernateRepository<TKey>` (`DataAccessLayer.Hibernate/IHibernateRepository.cs`), whose `UnitOfWork { get; }` property returns an `IHibernateUnitOfWork`. `HibernateRepository<TKey>` (same directory) implements this by lazily constructing a `HibernateUnitOfWork(_session)` on first access. Typical callers are repository consumers that want to wrap a series of reads/writes in a transaction with a chosen isolation level.

### Behavior & Details

- **`void BeginTransaction(IsolationLevel isolationLevel)`** — starts a new NHibernate transaction on the underlying session with the given `System.Data.IsolationLevel`. The inherited `BeginTransaction()` (no argument) starts one at the implementation's default level; this overload is for callers who need, for example, `IsolationLevel.RepeatableRead` or `IsolationLevel.Serializable`.
- The interface adds no other members. All transaction lifecycle concerns (`IsInTransaction`, `RollBackTransaction()`, `CommitTransaction()`, `Dispose()`) come from `IUnitOfWork`.
- The contract, as honored by the implementation, is that calling `BeginTransaction` (either overload) while a transaction is already active throws `InvalidOperationException`, and the transaction must be committed or rolled back before the unit of work is disposed. The interface itself is only a marker for the extra capability; the state rules live in `HibernateUnitOfWork`.
- Because the base `IUnitOfWork` already requires `Dispose()`, any class implementing `IHibernateUnitOfWork` must also implement disposal semantics; the NHibernate implementation disposes the underlying transaction but does not close or dispose the session itself (the session is owned by the repository or factory).