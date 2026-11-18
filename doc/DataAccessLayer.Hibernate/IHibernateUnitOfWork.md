# `DataAccessLayer.Hibernate/IHibernateUnitOfWork.cs`

### Purpose

This file defines `IHibernateUnitOfWork`, the NHibernate-specific unit-of-work contract. It exists so callers can depend on a way to control transactions with an explicit `System.Data.IsolationLevel`, which the generic base contract does not provide.

### Role in the Codebase

The interface lives in `DataAccessLayer.Hibernate` alongside its implementation `HibernateUnitOfWork` (see `HibernateUnitOfWork.cs`) and the repository types that expose it. It is a small extension point: it narrows the generic `IUnitOfWork` contract (defined in `DataAccessLayer/IUnitOfWork.cs`) for the NHibernate stack. If it were removed, `HibernateUnitOfWork` could no longer be consumed generically, and callers that need a non-default isolation level would have no typed way to ask for one.

### Relationships

- **Extends** `IUnitOfWork` from the `DataAccessLayer` namespace (`DataAccessLayer/IUnitOfWork.cs`). That base declares `IsInTransaction`, `BeginTransaction()`, `RollBackTransaction()`, `CommitTransaction()`, and (via `IDisposable`) disposal. Everything from the base is inherited by reference; this interface only adds one member.
- **Implemented by** `HibernateUnitOfWork`, which takes an NHibernate `IStatelessSession` and owns an `ITransaction`.
- **Exposed by** `HibernateRepository<TKey>` through its `UnitOfWork` property (declared on `IHibernateRepository<TKey>`), which lazily creates a `HibernateUnitOfWork` sharing the repository's session. So a repository is the natural entry point for obtaining the unit of work.
- **Consumed by business-layer code** — for example `BusinessLayer/UserService.cs` calls `repository.UnitOfWork.BeginTransaction()` before its operations.

### Behavior & Details

The single added member is:

```csharp
void BeginTransaction(IsolationLevel isolationLevel);
```

- It starts a transaction at the caller-specified `System.Data.IsolationLevel`. The base interface's parameterless `BeginTransaction()` remains available for the default case; `HibernateUnitOfWork`'s parameterless method simply delegates to this one with `IsolationLevel.ReadCommitted`, so this overload is the single source of truth for actually opening the transaction.
- The contract for the inherited members comes from `IUnitOfWork`: `IsInTransaction` reports whether a transaction is open, and `RollBackTransaction()`/`CommitTransaction()` end it. The implementation throws `InvalidOperationException` if you begin a second transaction before finishing the first, or try to commit/rollback with none running.
- Callers that need a stricter or looser isolation level than `ReadCommitted` (for example `Serializable` for a read-modify-write sequence) program against this interface rather than `IUnitOfWork` to reach the overload.

### Rough edges

- The base contract in `IUnitOfWork` has no way to express an isolation level, so every provider-specific unit-of-work interface in this codebase would have to add its own overload; there is no shared mechanism for it.