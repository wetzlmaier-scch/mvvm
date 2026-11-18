# `DataAccessLayer/IRepository.cs`

### Purpose

This interface defines the read-only query contract for repositories in the `DataAccessLayer` namespace. It exists so that the rest of the codebase can depend on a database-agnostic set of query operations rather than a specific ORM, letting `DataAccessLayer.Hibernate` supply the Hibernate-based implementation.

### Role in the Codebase

`IRepository<TKey>` sits at the core of the data access abstraction. It is implemented by `IHibernateRepository<TKey>` in `DataAccessLayer.Hibernate/IHibernateRepository.cs`, which extends it with write capability (`Save`) and a unit-of-work, so this interface is deliberately query-only. `IRepositoryFactory` in the same directory also constrains the repositories it hands out to this interface. If it were removed, the Hibernate layer and the factory contract would both lose their base contract.

Note the split in responsibilities: `IRepository<TKey>` covers reading, while `IHibernateRepository<TKey>` adds `Save<TEntity>` and `UnitOfWork`. Code that only reads entities can depend on `IRepository<TKey>` alone.

### Relationships

- **`IEntity<TKey>`** (same directory, `DataAccessLayer/IEntity.cs`): the marker interface every entity must implement — a single `TKey Id { get; }` property. Every method here constrains `TEntity` to `class, IEntity<TKey>` on top of it.
- **`DataAccessLayer.Hibernate.HibernateRepository<TKey>`**: the concrete implementation, reached through `IHibernateRepository<TKey>`, which adds `IDisposable` plus `Save` and `UnitOfWork` to this contract.
- **`DataAccessLayer.Hibernate.HibernateRepositoryFactory<TKey>`** and **`IRepositoryFactory`**: factories that produce repositories bound to this contract.
- **Callers**: `BusinessLayer/IRepositoryFactory.cs` wires `IHibernateRepositoryFactory<int>`, so the business layer is the layer that consumes repositories created from this contract.

### Behavior & Details

The generic parameter `TKey` (constrained to `struct`) is the identifier type for entities stored in this repository. The `in` (contravariant) variance declaration is unusual for an interface with no parameter-only positions; in practice it restricts assignability rather than adding flexibility, so do not rely on variance when substituting `TKey` types.

All seven members are constrained the same way: `TEntity : class, IEntity<TKey>`. There are no `TKey`-typed parameters on any member — `TKey` only flows through `TEntity`'s `Id`.

The methods group into three patterns:

- **Deferred queryables** — `GetQueryable<TEntity>(Expression<Func<TEntity, bool>> predicate)` and the parameterless `GetQueryable<TEntity>()`. These return `IQueryable<TEntity>`, so the expression tree is not evaluated at the call site; the consuming context (an active session in the Hibernate implementation) executes the query when it is enumerated or passed on to further LINQ operations. This is the escape hatch for callers who need to compose additional queries.
- **Single-result lookups** — `Single<TEntity>(criteria)`, `SingleOrDefault<TEntity>(criteria)`, and `FindOne<TEntity>(criteria)`. `Single` and `SingleOrDefault` follow the standard LINQ naming: `Single` returns `TEntity` (and throws if zero or more than one match), `SingleOrDefault` returns `TEntity?` (nullable, so it can represent "no match"). `FindOne` is a sibling of the same idea, also returning `TEntity?` — its distinction from `SingleOrDefault` is not encoded in the contract and lives in whichever implementation and calling convention the codebase uses.
- **Collection results** — `Find<TEntity>(criteria)` and `GetAll<TEntity>()`, both returning `IEnumerable<TEntity>`. `GetAll` has no predicate; `Find` narrows by expression.

Because the return types for `Find`, `GetAll`, and the `GetQueryable` overloads are `IQueryable`/`IEnumerable` rather than concrete lists, the interface does not commit to eager vs. deferred execution at this level — that depends on the implementation (the Hibernate-backed one) and on how the caller enumerates the result.

A few caveats worth knowing before you implement or extend this:

- No `IDisposable` here. `IHibernateRepository<TKey>` adds it, so code that holds a repository directly against `IRepository<TKey>` cannot dispose it — disposal is an `IHibernateRepository` concern.
- `TKey` is contravariant (`in`) despite never appearing in a parameter position. This is legal but almost certainly unintentional; treat it as if there were no variance modifier.
- `Single`/`SingleOrDefault`/`FindOne` overlap heavily; pick whichever name your implementation actually differentiates, rather than assuming the LINQ semantics for all three.

### Rough edges

- The `in` variance on `TKey` is unearned (no input positions) and may mislead readers about substitutability.
- `FindOne` and `SingleOrDefault` have near-identical signatures, and the interface gives no signal of how they differ in behaviour.
- No write path and no lifecycle (`IDisposable`) are part of this contract — a consumer holding only `IRepository<TKey>` cannot save or release resources; both live one layer down in `IHibernateRepository<TKey>`.