### Purpose

`DataAccessLayer/SortOrder.cs` defines `SortOrder`, a two-value enum (`Ascending`, `Descending`) that names the two possible sort directions for data queries. It exists so callers can express sort direction as a typed value rather than a string or boolean when ordering results through the data access layer.

### Role in the Codebase

It is a small supporting type within the `DataAccessLayer` namespace, alongside the repository and unit-of-work abstractions (`IRepository.cs`, `IUnitOfWork.cs`, etc.). It carries no logic of its own — its role is purely to give a shared, typed vocabulary for sort direction.

### Relationships

The type has no dependencies — it references nothing beyond the C# language itself. It is declared `public`, so any assembly that references `DataAccessLayer` can use it. As of this repository's content, no other file in `DataAccessLayer` (or anywhere else found in the repo) references `SortOrder` — the repository interfaces, for example, do not include it in their method signatures. It appears to be available for use by callers or future repository methods rather than consumed today.

### Behavior & Details

- `SortOrder` is a plain C# enum with two members: `Ascending` (value 0, the default) and `Descending` (value 1). No explicit values are assigned, so they take the implicit 0 and 1.
- Because `Ascending` is the default value, `default(SortOrder)` is `Ascending` — worth knowing if any consumer uses default-constructed parameters.
- There is no conversion logic, parsing, or formatting in this file.

### Rough edges

`SortOrder` is defined but referenced nowhere else in the repository. That is not a defect, but anyone adding a method that accepts it should be aware there is no existing precedent in this layer showing how it's meant to be threaded through queries (for instance, paired with a property-name or expression parameter).