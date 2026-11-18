# HibernateRepository.cs

**Path:** `DataAccessLayer.Hibernate/HibernateRepository.cs`

### Purpose

`HibernateRepository<TKey>` is the NHibernate-backed implementation of the repository pattern for this codebase. It gives callers LINQ query access to entities (all constrained to `IEntity<TKey>` so every entity has a strongly-typed key) plus a single `Save` for inserts, and it exposes a unit of work for transaction control. The NHibernate specifics (stateless sessions, no change tracking) are hidden behind the interface defined in `DataAccessLayer`.

### Role in the Codebase

It is the concrete workhorse of the `DataAccessLayer.Hibernate` assembly: the query side of `IHibernateRepository<TKey>` (which itself extends `DataAccessLayer.IRepository<TKey>` and `IDisposable`). It is not created directly by application code — `HibernateRepositoryFactory<TKey>.Create()` constructs it, passing in a new `IStatelessSession` from the factory's `ISessionFactory`. Each repository instance therefore owns one stateless session. If this class were removed, nothing could implement `IHibernateRepository<TKey>` and the whole Hibernate data access path would fall over.

### Relationships

- **`IHibernateRepository<TKey>`** (same directory): the contract implemented here. It adds `Save` and the `UnitOfWork` property on top of `IRepository<TKey>`.
- **`IRepository<TKey>`** (`DataAccessLayer/IRepository.cs`): declares the seven query members (`GetQueryable` x2, `Single`, `SingleOrDefault`, `Find`, `FindOne`, `GetAll`) that this class implements.
- **`IEntity<TKey>`** (`DataAccessLayer/IEntity.cs`): the entity marker interface requiring a `TKey Id`; every method's `TEntity` is constrained to it.
- **`HibernateUnitOfWork`** (same directory): lazily created through the `UnitOfWork` property; both share the same `IStatelessSession`.
- **NHibernate** (`IStatelessSession`, `NHibernate.Linq`): the external ORM the queries translate into.
- **Caller:** `HibernateRepositoryFactory<TKey>` is the only constructor call site. Consumers upstream of the data layer depend on it via the `IHibernateRepository<TKey>` / `IRepository<TKey>` interfaces.

### Behavior & Details

- **All reads are deferred.** `GetQuery<TEntity>()` returns `_session.Query<TEntity>()`, and every query method (`GetQueryable`, `Single`, `Find`, `FindOne`, etc.) just composes LINQ operators on that `IQueryable`. Nothing hits the database until the result is enumerated or materialized (`AsEnumerable`, `ToListAsync`, `FirstOrDefault`).
- **`GetAllAsync` uses EF-style `ToListAsync()`.** It is not from NHibernate — the project references an EF Core compatibility layer (`Microsoft.EntityFrameworkCore` or the `NHibernate.EFCore` package) that supplies `ToListAsync`. `GetAll` (sync) uses plain `AsEnumerable()` and executes the query on enumeration.
- **`Save` is insert-only.** It calls `_session.InsertAsync(entity)`, which matches stateless-session semantics: there are no updates or deletes anywhere in this repository.
- **The unit of work is shared, not private.** The `UnitOfWork` getter uses `_unitOfWork ??= new HibernateUnitOfWork(_session)`, so repeated access returns the same unit of work bound to this repository's session. Because `Save` executes on the session directly (no automatic transaction), any multi-statement work has to begin/commit a transaction explicitly through `UnitOfWork`.
- **`Dispose()` guards with `_disposed`** and disposes the lazily created unit of work if one was ever created. Note it never disposes `_session` itself — the session's lifecycle is the owner's responsibility.
- **`TKey : struct`** constraint on the class means keys must be value types (`int`, `Guid`, etc.); reference-type keys are not supported here.

### Rough edges

- `Dispose()` disposes the unit of work but not the `IStatelessSession` it wraps, and the interface contract (`IDisposable`) gives callers no hint that the session leaks if they don't manage it elsewhere.
- `Dispose()` checks `_disposed` but never re-arms, while it is marked `virtual` with no documented reason for overriding — overriding it could silently change the one-shot behavior.
- `FindOne` is `Where(criteria).FirstOrDefault()`, i.e. silently returns `null` when zero *or two or more* entities match, whereas `Single` throws on a count that is not exactly one. The naming can mislead a caller expecting "exactly one" semantics.
- The private `_unitOfWork` field is nullable with no null-forgiving usage elsewhere; `_unitOfWork` is assigned only in the property getter, so a `Dispose` before first access is a no-op — fine, but subtle.