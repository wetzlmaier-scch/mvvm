### Purpose

`DataAccessLayer.Hibernate/HibernateUnitOfWork.cs` defines `HibernateUnitOfWork`, the NHibernate-backed unit of work for this data access layer. It wraps a single `IStatelessSession`'s transaction lifecycle — begin, commit, rollback, dispose — so repository consumers can bracket their database operations in a transaction without dealing with NHibernate's `ITransaction` directly.

### Role in the Codebase

This is the concrete `IUnitOfWork` implementation for the NHibernate persistence layer, paired with the sibling `HibernateRepository<TKey>` in the same directory. The repository's `UnitOfWork` property creates it on demand: `_unitOfWork ??= new HibernateUnitOfWork(_session)` — i.e., each repository instance lazily creates one unit of work over the stateless session the factory (`HibernateRepositoryFactory<TKey>`) opened for it. If removed, `HibernateRepository`'s `UnitOfWork` property and `IUnitOfWork` would have no concrete implementation to hand out.

### Relationships

- **`IHibernateUnitOfWork`** (same directory): the interface this class implements. It extends `IUnitOfWork` (defined in `DataAccessLayer/IUnitOfWork.cs`, which declares `IsInTransaction`, `BeginTransaction()`, `RollBackTransaction()`, `CommitTransaction()`, and `IDisposable`) and adds one member, `BeginTransaction(IsolationLevel isolationLevel)`.
- **`IUnitOfWork`** (`DataAccessLayer/IUnitOfWork.cs`): the base interface; its members are implemented verbatim here.
- **`IDataAccessLayer.Hibernate.HibernateRepository<TKey>`**: the main caller — it owns a `readonly IStatelessSession` and passes it to the unit of work's constructor.
- **NHibernate** (external library): the `IStatelessSession` and `ITransaction` types and their `BeginTransaction`/`Commit`/`Rollback` semantics come from the NHibernate assembly, not this repository.

### Behavior & Details

- **Constructor** takes an `IStatelessSession` and stores it. No transaction is started at construction time; one must call `BeginTransaction` first.
- **`BeginTransaction()`** delegates to the overload with `IsolationLevel.ReadCommitted`.
- **`BeginTransaction(IsolationLevel)`** calls `_session.BeginTransaction(isolationLevel)` and stores the result in `_transaction`. It throws `InvalidOperationException` if a transaction is already running, so at most one transaction can be active per unit of work.
- **`CommitTransaction()`** commits, then releases (disposes and nulls) the transaction. If the commit throws, it rolls back and rethrows the original exception — the rollback here is defensive cleanup around a failed commit.
- **`RollBackTransaction()`** throws `InvalidOperationException` when no transaction is running; otherwise it rolls back and releases the transaction.
- **`IsInTransaction`** is simply `_transaction != null`, so it is true from the call to `BeginTransaction` until commit or rollback completes.
- **`Dispose()`** is virtual. Note that it only disposes the current transaction — it does not roll it back. A disposed unit of work with an uncommitted transaction therefore silently discards it; nothing is rolled back on the database. It also does not clear `_transaction` to null afterward, and `_isDisposed` is only set when a transaction was pending at disposal time.

### Rough edges

- The `CommitTransaction` pre-check error message says "Cannot roll back a transaction..." even though the method commits — a copy/paste leftover that will mislead anyone reading the exception.
- `Dispose()` does not set `_isDisposed` when no transaction is active, and it doesn't null `_transaction`, so `_isDisposed` and the actual state of the object are not reliably consistent.
- The class is not thread-safe, and `_transaction` has no `volatile`/locking; this is fine as long as each unit of work is confined to one thread of use, but nothing enforces that.