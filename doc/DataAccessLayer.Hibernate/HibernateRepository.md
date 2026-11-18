# Documentation

**File:** `DataAccessLayer.Hibernate/HibernateRepository.cs`

### Purpose

`HibernateRepository<TKey>` is the NHibernate-backed implementation of the repository pattern in this data access layer. It gives business-logic code LINQ-to-NHibernate queries plus a single insert operation over any entity type that implements `IEntity<TKey>`, and it exposes a unit of work for transaction control.

### Role in the Codebase

This class is the concrete repository consumed through `IHibernateRepository<TKey>` (defined in `DataAccessLayer.Hibernate/IHibernateRepository.cs`). Instances are created by `HibernateRepositoryFactory<TKey>.Create()`, which hands out one repository per `IStatelessSession` opened from the shared `ISessionFactory`. If this class were removed, the factory in `HibernateRepositoryFactory.cs` would have nothing to return, and any code typed against `IHibernateRepository<TKey>` or `IRepository<TKey>` would lose its NHibernate implementation.

### Relationships

- **`IHibernateRepository<TKey>`** (same directory): the direct interface implemented here. It extends `IDisposable` and `IRepository<TKey>` (from `DataAccessLayer/IRepository.cs`), which is why this class carries the query methods (`GetQueryable`, `Single`, `Find`, etc.) plus the `Save` method and `UnitOfWork` property that only the interface `IHibernateRepository` declares.
- **`IEntity<TKey>`** (`DataAccessLayer/IEntity.cs`): the entity marker interface requiring a struct `Id`. All generic methods here are constrained to `class, IEntity<TKey>`, so only mapped entities with an identity can be queried or saved.
- **`HibernateUnitOfWork`** (same directory): created lazily by the `UnitOfWork` property and wrapping the same `IStatelessSession`, so transaction begin/commit/rollback act on this repository's session.
- **`NHibernate` / `NHibernate.Linq`**: `IStatelessSession` for execution and the LINQ provider for translating the `IQueryable` chains.
- **Callers**: anything resolving `IHibernateRepository<TKey>` — e.g. code going through `IRepositoryFactory` (`BusinessLayer/IRepositoryFactory.cs`) which is rooted at `IHibernateRepositoryFactory<int>` — would rely on this class.

### Behavior & Details

- **Stateless session semantics.** The repository holds a single `IStatelessSession` for its lifetime. With stateless sessions NHibernate performs no identity map or dirty checking, which is why the class exposes only `InsertAsync` (in `Save<TEntity>`) for writes — there is no `Update` or `Delete`; updates must go through the session/`HibernateUnitOfWork` directly.

- **Query methods.** `GetQuery<TEntity>()` (private) wraps `_session.Query<TEntity>()`, the NHibernate.Linq entry point. Everything else is a thin LINQ expression over that:
  - `GetQueryable(predicate)` — filtered query, not yet executed.
  - `GetQueryable()` — unfiltered, as `IQueryable`.
  - `Single` / `SingleOrDefault` — execute immediately, throwing / returning null on zero-or-many / zero matches respectively.
  - `Find` — deferred filtered `IEnumerable`; `FindOne` — `FirstOrDefault`, so null on no match rather than an exception.
  - `GetAll` — deferred full-table enumeration; `GetAllAsync` — awaited materialization to a `List<TEntity>`.
  Queries are only sent to the database when enumerated, so `GetQueryable`/`Find`/`GetAll` return nothing until consumed.

- **`UnitOfWork` property.** Uses the null-coalescing assignment (`??=`) to create exactly one `HibernateUnitOfWork` per repository, always bound to this repository's session. Because `Dispose()` disposes `_unitOfWork`, disposing a repository also tears down any live unit of work.

- **`Dispose()`.** Guarded by `_disposed` so double-disposal is a no-op. It disposes the unit of work if one was created. Note it does **not** dispose `_session` — the session is owned by the factory's `OpenStatelessSession()` call, so callers disposing the repository leave the session open (see below).

- **`Save<TEntity>` is marked `async` but contains no `await`.** It simply calls `await _session.InsertAsync(entity)` — the method does actually await, so the warning only applies if you'd expect it to compose further; as written it performs one async insert.

### Rough edges

- `Dispose()` disposes the unit of work but never the underlying `IStatelessSession`, so disposing the repository can leak the session that `HibernateRepositoryFactory.Create()` opened for it.
- `GetAll<TEntity>` and `GetAllAsync<TEntity>` materialize the entire table with no paging, so they are risky on large entities.