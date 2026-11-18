# DataAccessLayer.Hibernate/IHibernateRepository.cs

### Purpose

This file defines `IHibernateRepository<TKey>`, the NHibernate-specific repository contract. It exists because the data layer's generic `IRepository<TKey>` is read-only, while business code also needs to persist entities through NHibernate and to control the surrounding transaction. This interface adds exactly that write path and transaction seam to the query contract.

### Role in the Codebase

It is one of the three interfaces in the `DataAccessLayer.Hibernate` module (`IHibernateRepository`, `IHibernateRepositoryFactory`, `IHibernateUnitOfWork`) that let consumers depend on an NHibernate-backed data layer without referencing NHibernate types. It sits at the boundary of the read/write split: `IRepository<TKey>` (`DataAccessLayer/IRepository.cs`) covers reading; this interface covers saving and lifecycle. If it were removed, the only implementor, `HibernateRepository<TKey>` (`DataAccessLayer.Hibernate/HibernateRepository.cs`), and the factory contract `IHibernateRepositoryFactory<TKey>` would have nothing to produce or implement, and the business layer's `IRepositoryFactory` (`BusinessLayer/IRepositoryFactory.cs`) chain would collapse.

### Relationships

- **Extends `IRepository<in TKey>`** (`DataAccessLayer/IRepository.cs`): inherited here are all the read operations — `GetQueryable`, `Single`, `SingleOrDefault`, `Find`, `FindOne`, `GetAll` — all generic over entities implementing `IEntity<TKey>`. This interface adds two things on top of those.
- **Extends `IDisposable`** (BCL): repositories hold an NHibernate session, so consumers must dispose them. Notably `IRepository<TKey>` is not disposable, so disposal is only possible when holding this contract.
- **Returns `IHibernateUnitOfWork`** (`DataAccessLayer.Hibernate/IHibernateUnitOfWork.cs`): the transaction unit exposed through the `UnitOfWork` property, which itself extends `IUnitOfWork` (`DataAccessLayer/IUnitOfWork.cs`) and adds `BeginTransaction(IsolationLevel)`.
- **Implemented by** `HibernateRepository<TKey>` (`DataAccessLayer.Hibernate/HibernateRepository.cs`), which the interface never names; consumers go through the abstraction.
- **Typical callers:** code resolving repositories through the factory chain — `IRepositoryFactory` in `BusinessLayer/IRepositoryFactory.cs` derives from `IHibernateRepositoryFactory<int>`, whose `Create()` returns an `IHibernateRepository<int>`.

### Behavior & Details

The interface declares one method and one property:

- **`Task Save<TEntity>(TEntity entity) where TEntity : class, IEntity<TKey>`** — the only write operation in the contract, and it is strictly an *insert*: the sole implementation calls `_session.InsertAsync(entity)` on the stateless session, so an existing row is not updated by this method. It returns a `Task`, so callers must await or observe it; the interface itself does not define any commit — a save is only durable when the associated unit of work commits (see the next point), since the stateless session does not auto-flush.
- **`IHibernateUnitOfWork UnitOfWork { get; }`** — exposes the transaction scope for the repository. The implementation creates the unit of work lazily on first access and reuses the same instance on subsequent reads; it shares the repository's stateless session, and disposing the repository disposes the unit of work with it. Callers who need a transaction get one from here rather than constructing one themselves.

`TKey` is constrained to `struct` and declared `in` (contravariant), matching `IRepository<TKey>`; in practice the codebase uses `int` keys.