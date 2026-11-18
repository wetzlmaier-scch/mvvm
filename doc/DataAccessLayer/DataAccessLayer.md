# DataAccessLayer.csproj

**Path:** `DataAccessLayer/DataAccessLayer.csproj`

### Purpose

This is the MSBuild project file (SDK-style `.csproj`) for the `DataAccessLayer` class library. It contains no application logic — it only declares how the C# source files in the `DataAccessLayer/` folder are compiled.

### Role in the Codebase

It is the build definition for a small contract/abstraction project that holds the data-access interfaces and support types: `IRepository.cs`, `IRepositoryFactory.cs`, `IUnitOfWork.cs`, `IEntity.cs`, `Entity.cs`, `SortOrder.cs`, and `DataAccessException.cs`. Notably, this project references no NuGet packages — it is a dependency-free set of abstractions (the only `using` in the folder is `System`, from `DataAccessException`). Concrete implementations live elsewhere, e.g. `DataAccessLayer.Hibernate`.

If it were removed, the `DataAccessLayer` assembly could not be built, and the two projects that reference it (below) would fail to build as well.

### Relationships

- **Built into:** the `DataAccessLayer` assembly, which is referenced by `DataAccessLayer.Hibernate` and `Entity.Hibernate` (via `<ProjectReference>` in their `.csproj` files) and included in the solution `Demo.slnx`.
- **Consumed by:** code in those two projects, which implement or extend the interfaces and types defined in this folder (e.g. `IRepository<T>`, `DataAccessException`).

### Behavior & Details

- `TargetFramework` is `net10.0`, so the library targets .NET 10.
- `ImplicitUsings` and `Nullable` are both enabled — this affects how the sibling source files are compiled (implicit `using` directives for common namespaces, and nullable reference-type annotations are on).
- There are no package references, no project references, and no custom build logic in this file; everything else in the folder compiles under these defaults.