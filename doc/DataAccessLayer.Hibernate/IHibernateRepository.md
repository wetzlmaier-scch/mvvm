### Purpose

`DataAccessLayer.Hibernate/IHibernateRepository.cs` defines `IHibernateRepository<TKey>`, the NHibernate-specific repository contract. It exists because the generic data-layer interface `IRepository<TKey>` is read-only, while business code also needs to persist entities and manage transactions. This interface is the seam: it layers the write side (`Save`, `UnitOfWork`) on top of the read side, so consumers can depend on persistence behavior without naming any NHibernate type.

It is one of the three NHibernate-specific interfaces in the `DataAccessLayer.Hibernate` folder (`IHibernateRepository`, `IHibernateRepositoryFactory`, `IHibernateUnitOfWork`) that let callers depend on an NHibernate-backed data layer without depending on the assembly itself.

### Role in the Codebase

This file is the central contract of the `DataAccessLayer.Hibernate` module. Its concrete implementation is `HibernateRepository<TKey>` in the same folder; instances are minted by `HibernateRepositoryFactory<TKey>` (via `IHibernateRepositoryFactory<TKey>`). The business layer reaches it through the factory chain: `BusinessLayer/IRepositoryFactory.cs` extends `IHibernateRepositoryFactory<int>`, whose `Create()` returns `IHibernateRepository<int>`.

If this interface were removed, `HibernateRepository<TKey>` would lose its public contract, the business layer would have no type to program its persistence calls against, and the read-only `IRepository<TKey>` would be the only repository surface available.

### Relationships

- **Extends `IRepository<TKey>`** (defined in `DataAccessLayer/IRepository.cs`): a read-only contract with `GetQueryable`, `Single`, `SingleOrDefault`, `Find`, `FindOne`, and `GetAll` over `IEntity<TKey>` entities. All of those members are inherited, not redeclared here.
- **Extends `IDisposable`**: repositories must be released by their callers, which is why business services (e.g. `UserService`) wrap each `Create()` call in a `using` block.
- **`Save<TEntity>(TEntity entity)`** — the one member this interface adds for writing. The entity constraint `class, IEntity<TKey>` means the type must expose a strongly-typed `Id` (per `DataAccessLayer/IEntity.cs`).
- **`IHibernateUnitOfWork UnitOfWork { get; }`** — exposes the NHibernate unit of work. `IHibernateUnitOfWork` (same folder) extends the base `DataAccessLayer.IUnitOfWork` (`IsInTransaction`, `BeginTransaction`, `RollBackTransaction`, `CommitTransaction`) and adds `BeginTransaction(IsolationLevel)`.
- **`where TKey : struct`** — repeated from the base constraint; it is what allows the business layer to close the generic to `int`.
- The `in` variance on `TKey` marks the repository as contravariant in its key type, matching the variance on the base `IRepository<TKey>`.

### Behavior & Details

The interface itself contains no logic — its behavior is defined by `HibernateRepository<TKey>`:

- `Save<TEntity>` is implemented as `await _session.InsertAsync(entity)` on the repository's NHibernate `IStatelessSession`. That means it always performs an `INSERT`; it does not distinguish entities that already exist, so callers must not use it to update.
- `UnitOfWork` is lazily initialized on first access (`_unitOfWork ??= new HibernateUnitOfWork(_session)`), so a read-only call pattern never pays for a unit of work.
- `Dispose()` disposes the unit of work if one was ever created, so transactional resources are cleaned up automatically when the repository is disposed.

Because `TEntity` is a per-call generic on both `Save` and the inherited read methods, a single repository instance can save and query many different entity types as long as they share the same `TKey`.