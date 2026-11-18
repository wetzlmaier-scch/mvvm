# BusinessLayer/BusinessLayer.csproj

### Purpose

This file is the MSBuild project file for the `BusinessLayer` class library. It defines how the layer compiles and what it links against, and nothing more — the actual business logic lives in the C# files alongside it (`UserService`, `RepositoryFactory`, `ModelContext`, and their interfaces).

### Role in the Codebase

It is one project in a solution that splits concerns by layer: the `BusinessLayer` folder holds the service and repository-factory code, and this `.csproj` is the unit the build system compiles. Two projects reference it, `UserManagement` and `TestProject1`, so removing the file removes the whole layer from the build for them.

### Relationships

- **References (via `ProjectReference`):** `DataAccessLayer.Hibernate`, `Dto`, `Entity.Hibernate`, and `Logging`. These match the namespaces the layer's code actually uses — `UserService` maps `Entity.Hibernate` entities to `Dto` types, `RepositoryFactory` and `ModelContext` derive from `DataAccessLayer.Hibernate` base types, and `IUserService` exposes a `Logging` service.
- **Referenced by:** `TestProject1/TestProject1.csproj` and `UserManagement/UserManagement.csproj`.
- **`InternalsVisibleTo` for `TestProject1`:** grants the test project access to internal members of this assembly, so tests can reach types that are not public.
- **NuGet package:** `CommunityToolkit.Diagnostics` (version 8.4.2), consumed by `UserService` for its `Guard` argument checks in `AddUser`.

### Behavior & Details

- **`<OutputType>Library</OutputType>`** — builds a class library, not an executable.
- **`<TargetFramework>net10.0</TargetFramework>`** — compiles for .NET 10.
- **`<ImplicitUsings>enable</ImplicitUsings>`** and **`<Nullable>enable</Nullable>`** — standard modern defaults: implicit `using` directives for common namespaces, and nullable-reference-type analysis on across the project. Note that `UserService`'s `[ImportingConstructor]` deliberately sidesteps the nullable warning with `LoggingService = null!;`, since the logging service is injected separately after construction rather than through the constructor.

### Rough edges

- The NuGet `PackageReference` for `CommunityToolkit.Diagnostics` pins a specific version (`8.4.2`). The package's `Guard` usage in `UserService` is the only consumer, but nothing in the project file guards against a future package removal leaving that import dangling — the compiler will catch it, but the dependency is easy to miss when editing this file.