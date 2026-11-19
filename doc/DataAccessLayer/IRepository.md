# `DataAccessLayer/IRepository.cs`

### Purpose

`IRepository<TKey>` is the storage-agnostic query contract for the data access layer. It lets business code read and filter entities by LINQ expression predicates without knowing that persistence is backed by NHibernate. It is read-only by design: no save, delete, or update operations live here — writes are handled by the provider-specific extension, `IHibernateRepository<TKey>`, which adds `Save` and a `UnitOfWork`.

The interface is keyed on `TKey` (the identifier type of the entities stored in that repository, constrained to `struct`) rather than on the entity type. Each method takes its entity type as its own generic parameter, so a single repository instance can query any entity whose `Id` is `TKey`.

### Role in the Codebase

This interface sits at the root of the `DataAccessLayer/` project, which holds the ORM-independent abstractions: `IEntity<TKey>` (the entity marker with an `Id`), the abstract base `Entity<TKey>`, this repository contract, and `IRepositoryFactory<TRepository, TKey>`. The concrete world lives one project over in `DataAccessLayer.Hibernate/`, where `HibernateRepository<TKey>` implements the NHibernate-specific contract and all query methods delegate to `IStatelessSession.Query<TEntity>()`.

`IRepository<TKey>` is the narrow surface that the business layer could in principle program against; in practice `BusinessLayer/UserService` receives the factory's `IHibernateRepository<int>` and calls `GetQueryable<T>()` on it, which is inherited from `IRepository<TKey>`. Removing this interface would break `IHibernateRepository<TKey>` (which extends it) and `IRepositoryFactory<TRepository, TKey>` (which constrains `TRepository : IRepository<TKey>`), collapsing the abstraction chain between `BusinessLayer` and NHibernate.

### Relationships

- **`IEntity<TKey>`** (defined in `DataAccessLayer/IEntity.cs`) — the constraint every `TEntity` must satisfy; it declares `TKey Id`. Every method here re-states the constraint `TEntity : class, IEntity<TKey>`.
- **`IHibernateRepository<TKey>`** (defined in `DataAccessLayer.Hibernate/IHibernateRepository.cs`) — the direct consumer; it extends `IRepository<TKey>` and `IDisposable` and adds `Save<TEntity>` and a `UnitOfWork` property.
- **`HibernateRepository<TKey>`** (defined in `DataAccessLayer.Hibernate/HibernateRepository.cs`) — the concrete implementation of every method on this interface, all backed by `IStatelessSession`.
- **`IRepositoryFactory<TRepository, TKey>`** (same directory) — constrains created repositories to `IRepository<TKey>`; its concrete descendant `IHibernateRepositoryFactory<TKey>` is what `BusinessLayer/RepositoryFactory` ultimately implements, and `UserService` obtains repositories through it.

### Behavior & Details

All seven members are query-only and generic over `TEntity : class, IEntity<TKey>`:

- `GetQueryable<TEntity>(predicate)` / `GetQueryable<TEntity>()` — return a deferred `IQueryable<TEntity>`, optionally pre-filtered with a LINQ expression. Because the result is `IQueryable`, callers can chain further `.Where`/`.Select`/`.ToListAsync()` etc.; in `UserService.GetActiveUsers` this is exactly how an in-memory projection to `UserDto` is performed against the session.
- `Single<TEntity>(criteria)` — evaluates immediately and returns exactly one matching entity. Like `Queryable.Single`, it throws if the match count is not exactly one.
- `SingleOrDefault<TEntity>(criteria)` — like `Single` but returns `null` when nothing matches; throws on more than one match.
- `Find<TEntity>(criteria)` — returns an `IEnumerable<TEntity>` of all matches (deferred enumeration). Note the name `Find` does not imply a single result; `FindOne` is the one-result variant.
- `FindOne<TEntity>(criteria)` — returns the first matching entity or `null`; never throws on zero matches.
- `GetAll<TEntity>()` — an `IEnumerable<TEntity>` over every entity of that type; in the Hibernate implementation it calls `AsEnumerable()`, so it is also lazily enumerated.

Two things the signatures do not say but the implementation does, and which a caller should know:

1. **`Find`, `GetAll`, and `GetQueryable` are lazy.** `HibernateRepository` implements them by returning `Where(criteria)` / `AsEnumerable()` / raw `Query<TEntity>()` without materializing. Consuming the sequence outside a live session (or without an explicit async LINQ-to-objects conversion such as `ToListAsync()`) can fail; `UserService` always terminates queries with `ToListAsync()`.
2. **`FindOne` vs `SingleOrDefault` are easy to conflate.** `FindOne` takes the *first* of any matches; `SingleOrDefault` *throws* when more than one match exists. Choose based on whether "at most one is expected" is an invariant you want enforced.

The interface itself is synchronous; the async variant (`GetAllAsync`) exists only on `HibernateRepository<TKey>`, not on this contract.

### Rough edges

- `TKey` is declared `in` (contravariant) on `IRepository<TKey>`, but `TKey` appears only in the `where` constraints of the generic methods, never in a parameter or return position of an interface member. The variance modifier therefore has no effect and misleads readers about how the type parameter is actually used.
- The constraint `TEntity : class, IEntity<TKey>` is repeated on all seven methods because it cannot be lifted to the interface declaration (an interface type parameter cannot be used as the generic argument of a constraint on a method type parameter). Concretely, the `Id` of `TEntity` must match the repository's `TKey`, which the compiler enforces via `IEntity<TKey>`; worth knowing if a generic entity base ever changes its `Id` type.