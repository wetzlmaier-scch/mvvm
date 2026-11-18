# `Entity.Hibernate/Entity.Hibernate.csproj`

### Purpose

This is the MSBuild project file for the `Entity.Hibernate` project — the small module in this solution that holds NHibernate-mapped domain entities. It declares that the project targets `net10.0` with implicit usings and nullable reference types enabled, and pulls in the two NuGet packages the entities need: `FluentNHibernate` 3.4.1 (for the `ClassMap` mapping style used in `UserMapping.cs`) and `NHibernate` 5.6.0 (the ORM runtime).

### Role in the Codebase

The project currently contains two files — `User.cs` (an entity deriving from `DataAccessLayer`'s `Entity<int>` base class) and `UserMapping.cs` (its Fluent NHibernate mapping to the `Users` table). This `.csproj` is what compiles those into a library that other projects can consume. Nothing about the mapping or entity design lives here; changing this file changes only build output, frameworks, and dependency versions.

### Relationships

- **`DataAccessLayer`** (referenced via `<ProjectReference Include="..\DataAccessLayer\DataAccessLayer.csproj" />`): provides the `Entity<TKey>` base class and repository/unit-of-work abstractions that `User` depends on.
- **NuGet packages**: `NHibernate` and `FluentNHibernate`, as above.
- **Callers**: any project in `Demo.slnx` that loads NHibernate mappings for the `Entity.Hibernate` namespace (e.g. a session-factory/mapping-registry assembly) would depend on this project's output.

### Behavior & Details

- `TargetFramework` is `net10.0`; the project does not multi-target.
- `Nullable` is enabled, which is why `User`'s protected constructor uses `null!` to silence nullability on `FirstName`/`LastName` before NHibernate hydrates them.
- No explicit `<AssemblyName>` or `<RootNamespace>` overrides, so the assembly and root namespace default to `Entity.Hibernate`.