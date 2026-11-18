# `DataAccessLayer/IUnitOfWork.cs`

### Purpose

`IUnitOfWork` is the provider-agnostic contract for managing database transactions. It lets callers begin, commit, and roll back a transaction without knowing which ORM is doing the actual work, so business logic can treat a sequence of data operations as an atomic unit. The file is part of `DataAccessLayer`, the project that holds the data-access abstractions with no logic of their own.

### Role in the Codebase

This interface is the transaction half of the `DataAccessLayer` contract set, alongside the read-oriented `IRepository<TKey>`, the `IRepositoryFactory`, and the entity types. It carries no implementation — it defines only the lifecycle a unit of work must support. If it were removed, the concrete NHibernate layer would lose its base contract: `IHibernateUnitOfWork` in `DataAccessLayer.Hibernate/IHibernateUnitOfWork.cs` extends it directly, and `HibernateUnitOfWork` implements it.

### Relationships

- **`System.IDisposable`** (standard .NET): `IUnitOfWork` extends it, so a unit of work is expected to release its underlying session/transaction resources when disposed.
- **`IHibernateUnitOfWork`** (`DataAccessLayer.Hibernate/IHibernateUnitOfWork.cs`): extends this interface and adds one member, `BeginTransaction(IsolationLevel)`, for isolation levels beyond the base contract.
- **`HibernateUnitOfWork`** (`DataAccessLayer.Hibernate/HibernateUnitOfWork.cs`): the concrete class that implements all of this interface's members, wrapping an NHibernate `IStatelessSession`.
- **Callers**: repositories expose the unit of work — `IHibernateRepository` has a `UnitOfWork` property — and business services use it. For example, `BusinessLayer/UserService.cs` (`AddUser`) calls `repository.UnitOfWork.BeginTransaction()`, commits on success, and rolls back on failure.

### Behavior & Details

The contract has exactly four members, and they define a simple state machine:

- **`bool IsInTransaction { get; }`** — reports whether a transaction is currently open.
- **`BeginTransaction()`** — opens a transaction. The implementation (`HibernateUnitOfWork`) starts with `ReadCommitted` isolation and throws `InvalidOperationException` if a transaction is already running, so at most one transaction is active at a time.
- **`CommitTransaction()`** — makes the open transaction permanent. The implementation throws `InvalidOperationException` if no transaction is running, and rolls back before rethrowing if the commit itself fails.
- **`RollBackTransaction()`** — discards the open transaction; also throws `InvalidOperationException` when there is no transaction to roll back.

Because the interface is provider-neutral, it intentionally says nothing about isolation levels — that is where the thinner `IHibernateUnitOfWork` extension comes in. Callers program against `IUnitOfWork` for the default behaviour and against `IHibernateUnitOfWork` when they need to choose an isolation level.

The standard usage pattern visible in the callers is: begin, do the work, commit; on any exception, roll back and let the caller handle the error.

### Rough edges

- The interface has no way to express an isolation level, so any provider-specific extension has to add its own `BeginTransaction(IsolationLevel)` overload; there is no shared place for that in the base contract.