# DataAccessLayer/IUnitOfWork.cs

## Purpose

`IUnitOfWork` is the provider-agnostic contract for managing database transactions in this codebase. It exists so that business logic (e.g., `UserService` in `BusinessLayer/`) can begin, commit, and roll back transactions without knowing which ORM or data access implementation is actually being used.

## Role in the Codebase

This file sits in the `DataAccessLayer` directory alongside other abstraction interfaces (`IRepository<TKey>`, `IRepositoryFactory`). It defines the transactional "unit" boundary for all data access: a caller starts a transaction, performs one or more repository operations, then commits or rolls back as a single atomic unit.

The concrete implementation is `HibernateUnitOfWork` (in `DataAccessLayer.Hibernate/`), which wraps an NHibernate `IStatelessSession`. A thinner sub-interface, `IHibernateUnitOfWork`, extends `IUnitOfWork` to add an overload of `BeginTransaction` that accepts an explicit `IsolationLevel`.

If this interface were removed, there would be no provider-agnostic way to express transactional boundaries, and callers would be forced to depend on the NHibernate-specific implementation directly.

## Relationships

- **Inherited by:** `IHibernateUnitOfWork` (DataAccessLayer.Hibernate/IHibernateUnitOfWork.cs) — adds `BeginTransaction(IsolationLevel isolationLevel)`.
- **Implemented by:** `HibernateUnitOfWork` (DataAccessLayer.Hibernate/HibernateUnitOfWork.cs) — the only concrete implementation in the repository.
- **Exposed through:** `IHibernateRepository.UnitOfWork` property, which returns an `IHibernateUnitOfWork`.
- **Consumed by:** `UserService` (BusinessLayer/UserService.cs) calls `repository.UnitOfWork.BeginTransaction()`, `CommitTransaction()`, and `RollBackTransaction()` to wrap user-save operations in a transaction.

`IUnitOfWork` itself extends `System.IDisposable` (standard .NET), so a unit of work is expected to release its underlying session/transaction resources when disposed.

## Behavior & Details

The interface declares four members:

| Member | Contract |
|---|---|
| `bool IsInTransaction { get; }` | Read-only flag: `true` when a transaction has been opened and not yet committed or rolled back. |
| `void BeginTransaction()` | Opens a new transaction. The `HibernateUnitOfWork` implementation starts with `IsolationLevel.ReadCommitted` by default and throws `InvalidOperationException` if a transaction is already open. |
| `void CommitTransaction()` | Commits the open transaction. Throws `InvalidOperationException` if no transaction is active. On failure, the implementation rolls back before rethrowing. |
| `void RollBackTransaction()` | Rolls back the open transaction. Throws `InvalidOperationException` if no transaction is active. |

### Typical usage pattern

```csharp
using var repository = _repositoryFactory.Create();
repository.UnitOfWork.BeginTransaction();
try
{
    await repository.Save(entity);
    repository.UnitOfWork.CommitTransaction();
}
catch
{
    repository.UnitOfWork.RollBackTransaction();
    throw;
}
```

Callers (such as `UserService.AddUser`) are responsible for the try/catch and for deciding whether to rethrow or wrap the exception. The unit of work itself does not swallow errors.

### Rough edges

- The `CommitTransaction` failure path in `HibernateUnitOfWork` calls `RollBackTransaction()`, but if that rollback also throws, the original exception is lost (the rollback exception propagates instead). This is an implementation detail outside this interface, but it is relevant to anyone relying on the "commit or throw" contract.
- `Dispose` on `HibernateUnitOfWork` sets `_isDisposed = true` only when a transaction existed at disposal time, which means calling `Dispose` twice with no transaction open leaves the flag unset. Again, this lives in the implementation, not the contract.