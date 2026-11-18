# Documentation

### Purpose

`DataAccessLayer.Hibernate/HibernateUnitOfWork.cs` defines `HibernateUnitOfWork`, the NHibernate-backed unit of work for this data access layer. It wraps the transaction lifetime of a single `IStatelessSession`: begin, commit, rollback, and dispose of the transaction while guarding against invalid state transitions. Business code that persists through `HibernateRepository<TKey>` uses it to delimit unit-of-work transactions around a group of operations.

### Role in the Codebase

It is the concrete implementation of `IUnitOfWork` (defined in `DataAccessLayer/IUnitOfWork.cs`) and `IHibernateUnitOfWork` (defined in `DataAccessLayer.Hibernate/IHibernateUnitOfWork.cs`). The main user is `HibernateRepository<TKey>` (same directory): its `UnitOfWork` property lazily creates a `HibernateUnitOfWork` and shares the repository's own `IStatelessSession` with it, so a transaction begun here scopes the session the repository's queries and `Save` calls run on. Disposing a repository disposes this unit of work via `HibernateRepository.Dispose()`.

### Relationships

- **Base interfaces, both in this codebase:**
  - `IUnitOfWork` (in `DataAccessLayer/IUnitOfWork.cs`) declares `IsInTransaction`, `BeginTransaction()`, `RollBackTransaction()`, `CommitTransaction()`, and inherits `IDisposable` from `System.IDisposable`. This class provides all of them.
  - `IHibernateUnitOfWork` (in `DataAccessLayer.Hibernate/IHibernateUnitOfWork.cs`) extends `IUnitOfWork` with one extra member, `BeginTransaction(IsolationLevel isolationLevel)`. This class is the only implementation of either interface.
- **NHibernate (external library):** `IStatelessSession`, `ITransaction`, and their `BeginTransaction`/`Commit`/`Rollback` semantics come from the NHibernate assembly.
- **Callers:** `HibernateRepository<TKey>` constructs it in its `UnitOfWork` property getter and disposes it in its own `Dispose()`. Anything that holds an `IUnitOfWork` or `IHibernateUnitOfWork` (typically obtained via `repository.UnitOfWork`) can drive the transaction without knowing this concrete type.

### Behavior & Details

- **Construction** takes an `IStatelessSession` and stores it as the sole field. No transaction is started at construction; a caller must invoke `BeginTransaction` before committing or rolling back.

- **`IsInTransaction`** returns true exactly while an `ITransaction` instance is held. It is the guard used by every method below, so callers can check it to decide whether a transaction is active.

- **`BeginTransaction()`** delegates to the overload with `IsolationLevel.ReadCommitted`, the SQL Server default.

- **`BeginTransaction(IsolationLevel)`** calls `_session.BeginTransaction(isolationLevel)` and stores the returned `ITransaction`. It throws `InvalidOperationException` if a transaction is already active, so there is no support for nested transactions on one session — the running transaction must be committed or rolled back first.

- **`CommitTransaction()`** commits the active transaction, then releases it (dispose and null out the field, see `ReleaseCurrentTransaction`). If the commit throws, it calls `RollBackTransaction()` and then rethrows the original exception. Note the rollback here is best-effort cleanup on the failure path.

- **`RollBackTransaction()`** rolls back the active transaction, then releases it in the same way. Throws `InvalidOperationException` when no transaction is active.

- **`ReleaseCurrentTransaction()`** is the private helper that disposes the `ITransaction` and sets the field to null, which is what flips `IsInTransaction` back to false.

- **`Dispose()`** disposes the `ITransaction` if one is still open at disposal time and sets `_isDisposed`. Note that it does *not* dispose the underlying `IStatelessSession` — that object's lifetime is owned by whoever constructed the repository (see `HibernateRepositoryFactory`), and the repository's `Dispose()` likewise never disposes its session. So a transaction still open when the repository is disposed is left without a commit or rollback; it is simply disposed.

- **`_isDisposed`** is set only inside `Dispose()` and is never reset; the class has no `GetService`-style reentrancy guard beyond the null check on `_transaction`. Calling `Dispose()` twice is safe because the second call sees `_transaction` as null after `ReleaseCurrentTransaction` already cleared it, or because `_isDisposed` is already true.

### Rough edges

- The error message in `CommitTransaction()` reads "Cannot roll back a transaction while there is no transaction running." even though the failing operation is a commit. This is a copy-paste of the rollback guard's message and will mislead anyone logging or surfacing the exception.
- `_isDisposed` is assigned but never read in a way that changes behaviour — the only guard that matters is the `_transaction != null` check. It is dead state that suggests a dispose-guard pattern this class does not actually enforce.
- `Dispose()` does not roll back or commit an open transaction; it just disposes it. Callers relying on `using` blocks to clean up uncommitted work should know the transaction is silently dropped.
- `BeginTransaction` has no overload for `IsolationLevel.Unspecified` or for ambient transactions; the only supported levels are whatever `IsolationLevel` values NHibernate's `IStatelessSession.BeginTransaction` accepts.