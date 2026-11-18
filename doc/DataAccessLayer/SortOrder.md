### Purpose

`DataAccessLayer/SortOrder.cs` defines a single public enum, `SortOrder`, with two members — `Ascending` and `Descending` — expressing the direction in which query results should be sorted. It exists as a shared, strongly-typed vocabulary for sort direction instead of call sites using raw strings or magic integers.

### Role in the Codebase

The file lives in `DataAccessLayer`, a small contract-and-abstraction project holding the data-access interfaces (`IRepository.cs`, `IRepositoryFactory.cs`, `IUnitOfWork.cs`) and entity types (`IEntity.cs`, `Entity.cs`) plus the shared exception `DataAccessException.cs`. `SortOrder` is a support type alongside those contracts: it names a concept the data-access layer cares about, sort direction, so any future query API in this project can reference one definition.

### Relationships

It has no imports and no dependencies — it is a plain C# enum with no base types or attributes. It sits in the `DataAccessLayer` namespace, so consumers would reference it as `DataAccessLayer.SortOrder.Descending` or with a `using DataAccessLayer;` directive.

### Behavior & Details

- The single line `public enum SortOrder { Ascending, Descending }` is an un-decorated C# enum, so the members get the default underlying `int` values: `Ascending = 0`, `Descending = 1`. `Ascending` is therefore also the implicit default when a `SortOrder` parameter is initialized to zero.
- The enum has no current callers anywhere in the repository. In particular, `IRepository<T>` in `IRepository.cs` exposes no ordering parameter — its members accept predicates and return `IQueryable`/collections with no `SortOrder` involved. It is also not accepted by `IRepositoryFactory` or `IUnitOfWork`. Its usage is implied only by its name and location.

### Rough edges

`SortOrder` is defined but not referenced by any other type in the project, so a reader cannot tell from the code which API is meant to accept it.