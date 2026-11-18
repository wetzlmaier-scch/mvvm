# DataAccessLayer/SortOrder.cs

### Purpose

`DataAccessLayer/SortOrder.cs` defines a small public enum, `SortOrder`, with two members — `Ascending` and `Descending` — intended to express the direction in which query results should be sorted. It is a shared vocabulary for the data access layer: code that issues queries can refer to a sort direction by this type rather than by strings or booleans.

### Role in the Codebase

It is a leaf data type in the `DataAccessLayer` project, which otherwise holds the persistence abstractions (`IEntity<TKey>`, `Entity<TKey>`, `IRepository<TKey>`, `IRepositoryFactory`, `IUnitOfWork`) plus `DataAccessException`. This enum is the only sort-related type in the project. It has no logic of its own and no dependencies.

### Relationships

- No usings and no references to other code in the repository; it is fully self-contained.
- Nothing in the repository currently uses it. The `IRepository<TKey>` interface in `IRepository.cs` exposes query methods (`GetQueryable`, `Find`, `GetAll`, …) that return unordered `IQueryable`/`IEnumerable` results and take no ordering parameter; `IHibernateRepository<TKey>` in `DataAccessLayer.Hibernate` adds only `Save` and a `UnitOfWork`. So the enum is presently a definition awaiting consumers rather than part of a live code path.
- Removing it would break compilation of any project that references it, but today nothing does.

### Behavior & Details

- `public enum SortOrder { Ascending, Descending }` — an un-decorated C# enum, so the members get the default underlying `int` values: `Ascending = 0`, `Descending = 1`. `Ascending` is therefore also the value `default(SortOrder)` resolves to.
- Because it is in the `DataAccessLayer` namespace, consumers can use it as `DataAccessLayer.SortOrder.Descending` or with a `using DataAccessLayer;` directive.

### Rough edges

The enum has no current callers in the repository — no interface, repository implementation, or query API accepts a `SortOrder` — so its intended usage is only implied by its name and location.