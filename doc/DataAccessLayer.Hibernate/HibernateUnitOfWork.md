# `DataAccessLayer.Hibernate/HibernateUnitOfWork.cs`

### Purpose

`HibernateUnitOfWork` is the NHibernate-specific unit of work for the data access layer: it wraps a single `IStatelessSession` transaction so business code can begin, commit, and roll back work without touching NHibernate types directly. It implements the provider-agnostic `IUnitOfWork` contract (in `DataAccessLayer/IUnitOfWork.cs`) plus the NHibernate-only `IHibernateUnitOfWork` (same directory), which adds `BeginTransaction(IsolationLevel)` so callers can choose the isolation level instead of being stuck with the default.

### Role in the Codebase

It is the transaction primitive of the `DataAccessLayer.Hibernate` project. Repositories are the ones that actually perform reads and writes; this class is how they — or anything else holding a reference — group those operations into one atomic transaction. The primary in-repo caller is `HibernateRepository<TKey>` (in `HibernateRepository.cs`), which lazily creates one per repository instance via its `UnitOfWork` property: `_unitOfWork ??= new HibernateUnitOfWork(_session)`. If removed, the NHibernate side of the data access layer would have no way to run multi-statement work atomically.

### Relationships

- **`IUnitOfWork`** (`DataAccessLayer/IUnitOfWork.cs`): the base contract. Declares `IsInTransaction`, `BeginTransaction()`, `RollBackTransaction()`, `CommitTransaction()`, and extends `IDisposable`. Everything except the isolation-level overload comes from this interface.
- **`IHibernateUnitOfWork`** (`DataAccessLayer.Hibernate/IHibernateUnitOfWork.cs`): extends `IUnitOfWork` with `BeginTransaction(IsolationLevel isolationLevel)`. This class is the only implementor of that interface.
- **`IStatelessSession`** (NHibernate): the session passed in via the constructor. Note the session is *not* created or disposed by this class — the owning repository holds it for its lifetime, so several unit-of-work instances can share one session.
- **`HibernateRepository<TKey>`** (`HibernateRepository.cs`): constructs a `HibernateUnitOfWork` sharing its own session and disposes it in its `Dispose()`.

### Behavior & Details

- **Single transaction at a time.** Exactly one `ITransaction` may be active. `_transaction != null` drives the `IsInTransaction` property, and `BeginTransaction` throws `InvalidOperationException` if a transaction is already running — you cannot nest or stack transactions here.

- **`BeginTransaction()`** starts with `IsolationLevel.ReadCommitted`; the overload takes the level explicitly and hands it straight to `_session.BeginTransaction(isolationLevel)`.

- **`CommitTransaction()`** calls `_transaction.Commit()` inside a try/catch. If the commit throws (e.g. a constraint violation surfaces at commit time), it calls `RollBackTransaction()` and then rethrows the original exception. In practice NHibernate has usually already marked the transaction as needing rollback, so the inner `Rollback()` typically does nothing, but the release of the transaction reference still happens.

- **`RollBackTransaction()`** throws `InvalidOperationException` if no transaction is running, otherwise rolls back and clears the reference.

- **`Dispose()`** has two quirks worth knowing:
  - It only disposes the transaction — the session is never touched (correct, since the repository owns it).
  - It does **not** roll back a live transaction on dispose; the in-flight transaction is simply abandoned without commit or rollback. Also, `_isDisposed` is only set when a transaction is actually present, and nothing is guarded by it afterwards — repeated `Dispose()` calls re-dispose the transaction object.

- The commit path releases the transaction via `ReleaseCurrentTransaction()`, which disposes the `ITransaction` and sets `_transaction = null`, so `IsInTransaction` becomes false only after a successful commit or a rollback.

### Rough edges

- `CommitTransaction()` uses the wrong error message for the no-active-transaction check — it says "Cannot roll back a transaction" instead of commit.
- `Dispose()` neither rolls back nor commits a live transaction, and its `_isDisposed` guard is ineffective (only set when a transaction exists, never consulted again), so double-dispose re-disposes the underlying `ITransaction`.