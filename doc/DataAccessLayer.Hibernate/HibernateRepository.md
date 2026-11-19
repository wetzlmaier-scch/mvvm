# `DataAccessLayer.Hibernate/HibernateRepository.cs`

### Purpose

`HibernateRepository<TKey>` is the NHibernate-backed implementation of the repository pattern for this codebase. It gives callers LINQ query access to entities (all constrained to `IEntity<TKey>`) and a single write operation — `Save` — built on a *stateless* session, which is the source of several behaviours below.

It exists so business-layer code can depend on the `IRepository<TKey>` contract in the `DataAccessLayer` project rather than on NHibernate directly.

### Role in the Codebase

This class is the concrete leaf of the persistence stack in `DataAccessLayer.Hibernate`. The flow is:

- `HibernateRepositoryFactory<TKey>` (same directory) builds the `ISessionFactory` and, in `Create()`, opens a stateless session and hands it to `new HibernateRepository<TKey>(...)`. This is the only construction site in the repository.
- Callers work through the `IHibernateRepository<TKey>` interface, so they never name this class directly.
- `IHibernateUnitOfWork` / `HibernateUnitOfWork` (same directory) are lazily created by the `UnitOfWork` property for callers that need explicit transaction control around their operations.

If this class were removed, `HibernateRepositoryFactory.Create()` would have nothing to return, and all repository access through the NHibernate stack would be gone.

### Relationships

- **Implements `IHibernateRepository<TKey>`** (`DataAccessLayer.Hibernate/IHibernateRepository.cs`), which itself extends `IRepository<TKey>` (`DataAccessLayer/IRepository.cs`) and `IDisposable`. All the read methods (`GetQueryable`, `Single`, `SingleOrDefault`, `Find`, `FindOne`, `GetAll`) come from `IRepository<TKey>`; this class adds `Save` and the `UnitOfWork` property.
- **`IEntity<TKey>`** (`DataAccessLayer/IEntity.cs`): the entity marker interface requiring a `TKey Id`; every method's `TEntity` is constrained to it.
- **NHibernate (external library)**: `IStatelessSession` is the injected session; `NHibernate.Linq` provides the `Query<TEntity>()` and `ToListAsync()` extension methods.
- **`HibernateUnitOfWork`** (same directory): instantiated lazily by the `UnitOfWork` property using the repository's own session.

### Behavior & Details

- **Stateless session, no identity map.** The repository holds an `IStatelessSession`, not a regular `ISession`. Queries work normally through `_session.Query<TEntity>()`, but there is no object cache, no dirty checking, and no `Update`/`Delete` — which is why the write surface is exactly one method:

  ```csharp
  public async Task Save<TEntity>(TEntity entity) where TEntity : class, IEntity<TKey>
  {
      await _session.InsertAsync(entity);
  }
  ```

  Despite the name, `Save` is always an **insert**; there is no upsert. The `async` method has no `await` (it simply returns the completed `InsertAsync` task).

- **Read methods are thin LINQ pass-throughs.** `GetQuery<TEntity>()` (private) centralises `_session.Query<TEntity>()`, and every read method just composes it: `Find` applies a `Where`, `FindOne` adds `FirstOrDefault()`, `GetAll` forces enumeration with `AsEnumerable()` (synchronous), `GetAllAsync` materialises to a `List<TEntity>` via `ToListAsync()`. `Single`/`SingleOrDefault` delegate straight to the LINQ operators, so they throw when zero or multiple rows match, respectively.

- **`UnitOfWork` is lazily created and shared per repository instance.**

  ```csharp
  public IHibernateUnitOfWork UnitOfWork
  {
      get { return _unitOfWork ??= new HibernateUnitOfWork(_session); }
  }
  ```

  The same `HibernateUnitOfWork` (and thus the same session) is returned on every access. Transaction semantics — one active transaction at a time, `CommitTransaction` rolling back and rethrowing on failure — live in `HibernateUnitOfWork`, not here.

- **`Dispose()` and the `_unitOfWork` field.** The field is never assigned anywhere outside the `Dispose()` check:

  ```csharp
  if (_unitOfWork != null)
      _unitOfWork.Dispose();
  ```

  Because the `UnitOfWork` property writes to a *property* that holds the instance in `_unitOfWork` — actually it does; `??=` writes to `_unitOfWork` — so the null check in `Dispose()` is real: the unit of work is disposed only if `UnitOfWork` was ever accessed. The session itself is never disposed by this class; ownership of the session belongs to the factory that opened it.

- **`_disposed` flag.** Guards double disposal, but disposal is idempotent on the unit of work anyway (`HibernateUnitOfWork.Dispose` has its own guard).

### Rough edges

- `Dispose()` never disposes the `IStatelessSession` it was given. The session is created by `HibernateRepositoryFactory.Create()` and no caller in the repository disposes it, so each repository instance leaks an open stateless session unless NHibernate closes it out-of-band.
- The error message in `HibernateUnitOfWork.CommitTransaction` says "Cannot roll back" when it should say "Cannot commit" — a copy/paste slip that will mislead anyone reading that exception.
- `Save` is named generically but is insert-only; a caller wanting to update an existing row has no method available on this repository.