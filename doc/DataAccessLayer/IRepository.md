# DataAccessLayer/IRepository.cs

### Purpose

`DataAccessLayer/IRepository.cs` defines `IRepository<TKey>`, the key-based, ORM-agnostic read contract for the data access layer. Business code queries entities through this interface without knowing which persistence engine sits behind it. It deliberately covers only read operations — finding, filtering, and projecting entities — so the same contract stays true regardless of how an implementation handles writes, transactions, or sessions.

It exists alongside sibling contracts (`IRepositoryFactory`, `IUnitOfWork`) in the interface-only `DataAccessLayer` project, keeping the persistence engine out of the type signatures that higher layers depend on.

### Role in the Codebase

This file is part of the `DataAccessLayer` contract project, which contains only abstractions: `IRepository<TKey>`, `IEntity<TKey>`, `Entity<TKey>`, `IRepositoryFactory`, `IUnitOfWork`, and the `SortOrder` enum. It is the central query contract of that project.

`IRepository<TKey>` is generic over the identity key type and is constrained `where TKey : struct`. Every concrete repository in the codebase works with an `int` key (through `IHibernateRepository<int>`), so in practice `TKey` is always `int`. The generic parameter exists so the contract does not hard-code that assumption.

If this interface were removed, `IHibernateRepository<TKey>` (in `DataAccessLayer.Hibernate`) would have nothing to extend, and all business-layer query calls typed against `IRepository<TKey>` would lose their contract. The abstraction that lets `BusinessLayer` stay NHibernate-free is defined right here.

### Relationships

- **`IEntity<TKey>`** (same project, `DataAccessLayer/IEntity.cs`): the marker interface a single `TKey Id { get; }` property. Every method on `IRepository<TKey>` constrains `TEntity : class, IEntity<TKey>`, so only entity types carrying a matching identity can be queried. Note there are no `TKey`-typed parameters on any member — `TKey` flows exclusively through `TEntity`'s `Id`.
- **`IHibernateRepository<TKey>`** (`DataAccessLayer.Hibernate/IHibernateRepository.cs`): the NHibernate-specific contract. It extends both `IDisposable` and `IRepository<TKey>`, adding write/transaction capabilities (`Save<TEntity>`, a `UnitOfWork` property) on top of this read-only surface.
- **`HibernateRepository<TKey>`** (`DataAccessLayer.Hibernate/HibernateRepository.cs`): the concrete implementation, created by `HibernateRepositoryFactory<TKey>` over an `IStatelessSession`. It implements every member of this interface against NHibernate's LINQ provider (`session.Query<TEntity>()`).
- **Callers**: business services never name this interface directly in most cases — they go through `IRepositoryFactory` (`BusinessLayer/IRepositoryFactory.cs`), which returns `IHibernateRepository<int>` (and therefore `IRepository<int>`). `BusinessLayer/UserService` is the representative caller: it does `using var repository = _repositoryFactory.Create();` per operation and then calls `repository.GetQueryable<User>()`.

### Behavior & Details

All seven members are constrained identically — `where TEntity : class, IEntity<TKey>` — and all are query methods with no side effects at the contract level. The variance modifier `in TKey` makes the interface contravariant in its key, matching how it is meant to be substituted.

The members fall into three shapes:

- **`IQueryable<TEntity>` access** — the most flexible surface:
  - `GetQueryable<TEntity>()` — the full, unfiltered queryable for the entity type. Callers can apply any LINQ (`Where`, `Select`, `OrderBy`, `ToListAsync`, etc.).
  - `GetQueryable<TEntity>(Expression<Func<TEntity, bool>> predicate)` — the same, pre-filtered by a predicate.
- **Materialized single results**:
  - `Single<TEntity>(criteria)` — exactly one match; per LINQ semantics it throws if zero or more than one entity matches.
  - `SingleOrDefault<TEntity>(criteria)` — zero or one match; returns `null` if none.
  - `FindOne<TEntity>(criteria)` — effectively `FirstOrDefault`: returns the first match or `null`, and does **not** throw if more than one entity matches. This is the semantic difference between `FindOne` and `Single`/`SingleOrDefault`.
- **Materialized collections**:
  - `Find<TEntity>(criteria)` — all entities matching the predicate.
  - `GetAll<TEntity>()` — every entity of that type.

Because `Single` and `FindOne` return `TEntity?` while `Single` returns `TEntity` (non-nullable in signature), a caller using `FindOne` or `SingleOrDefault` must be prepared for `null`.

There are no async members on this interface, and no ordering/sorting parameter (the project's `SortOrder` enum is not referenced here) — callers needing ordered results compose `OrderBy` on the `IQueryable` they obtain from `GetQueryable`.

### Rough edges

- `Find` and `FindOne` overlap in purpose with `GetQueryable(predicate)` and `GetQueryable(predicate).FirstOrDefault()`; the only difference is that the `Find*` methods return `IEnumerable`/materialized values rather than a queryable. The near-synonymous names (`FindOne` vs `SingleOrDefault` vs `Single`) can mislead a caller about multiplicity behavior — only `Single` enforces "exactly one."