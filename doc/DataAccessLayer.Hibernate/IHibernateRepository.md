# DataAccessLayer.Hibernate/IHibernateRepository.cs

### Purpose

`DataAccessLayer.Hibernate/IHibernateRepository.cs` defines `IHibernateRepository<TKey>`, the NHibernate-specific contract that extends the data layer's generic `IRepository<TKey>` (`DataAccessLayer/IRepository.cs`) with two Hibernate concerns: an explicit `Save<TEntity>` method and a `UnitOfWork` property that exposes the unit of work backed by the same session.

### Role in the Codebase

The generic `IRepository<TKey>` in the `DataAccessLayer` namespace defines only query operations (`GetQueryable`, `Single`, `Find`, `GetAll`, and so on). Callers who want to persist through NHibernate need an interface that also declares saving and transaction access; that is this interface. It is implemented by `HibernateRepository<TKey>` in the same directory (`DataAccessLayer.Hibernate/HibernateRepository.cs`), and produced by factories implementing `IHibernateRepositoryFactory<TKey>` (`DataAccessLayer.Hibernate/IHibernateRepositoryFactory.cs`). Domain/application code can depend on `IHibernateRepository<TKey>` rather than the concrete class, or on plain `IRepository<TKey>` when it only needs reads.

### Relationships

- Inherits `IRepository<TKey>` from `DataAccessLayer/IRepository.cs` — all read/query members come from that interface and are not restated here.
- Inherits `IDisposable` from `System`.
- `IEntity<TKey>` (referenced in the `Save` constraint) is defined in `DataAccessLayer/IEntity.cs` as a marker requiring a `TKey Id` property; abstract base class `Entity<TKey>` in `DataAccessLayer/Entity.cs` implements it.
- `IHibernateUnitOfWork` is defined in `DataAccessLayer.Hibernate/IHibernateUnitOfWork.cs` and implemented by `HibernateUnitOfWork` in the same directory.

### Behavior & Details

- `TKey` is constrained to `struct` and marked `in` (contravariant), so a repository typed on a derived key can be used where a base-key repository is expected.
- `Save<TEntity>(TEntity entity)` is the only persistence operation declared. In the sole implementation, `HibernateRepository.Save`, it calls `IStatelessSession.InsertAsync(entity)`. Because the session is a *stateless* NHibernate session, `Save` is an immediate insert — there is no identity map or dirty checking. If an entity already exists in the database, a second `Save` will produce a duplicate-key error; this interface has no `Update` or `Delete` members at all.
- `UnitOfWork { get; }` exposes the `IHibernateUnitOfWork` bound to the repository's session. The implementation lazily creates a `HibernateUnitOfWork(_session)` on first access, so the caller controls transaction scope (`BeginTransaction`, `CommitTransaction`, `RollBackTransaction`) around any work performed through that repository. `Save` executes within the active transaction on that session.
- `Dispose` (inherited from `IDisposable`) disposes the associated unit of work, and by extension any uncommitted transaction on it.

### Rough edges

The `UnitOfWork` property on the *interface* is a lazy, mutable cache in the implementation: once a unit of work is created for a repository, the repository effectively has a committed scope of work for its lifetime, which can surprise callers who obtain several repositories from the factory expecting independent transactions.