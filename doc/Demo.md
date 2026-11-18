# Demo.slnx

**Path:** `Demo.slnx` (repository root)

This is a .NET solution file in the XML-based `.slnx` format. It defines the solution that groups all 16 projects in this repository so they can be built, opened in Visual Studio or VS Code, and built together.

### Purpose

The solution file exists so the repo's many small, layered projects can be opened and built as a single unit instead of one at a time. It carries no logic of its own — its only job is the project list (plus, where present, a GUID per project, used for stable identification across renames and moves).

### Role in the Codebase

This is the top-level entry point for the whole repository. Every project directory at the root — `BusinessLayer`, `Common`, `Mvvm`, `DataAccessLayer`, `UserManagement`, the three UI hosts, and the test project — is registered exactly once in this file. All 16 registered `Path` entries match the 16 `*.csproj` files that exist, so there are no stale or orphaned entries. If the solution file were removed, each project could still be built on its own from its own `.csproj` (project references in the `.csproj` files are what carry the real dependency graph), but you would lose the single-window build/open experience.

The projects in the solution group into a few layers, and the reference graph in the `.csproj` files shows how they stack:

- **Core libraries:** `Common`, `Dto`, `Logging`, `Mvvm`, `DataAccessLayer`, `Entity.Hibernate` (Hibernate ORM entities), `DataAccessLayer.Hibernate` (persistence, built on `Entity.Hibernate`).
- **Business logic:** `BusinessLayer` references `DataAccessLayer.Hibernate`, `Dto`, `Entity.Hibernate`, and `Logging`.
- **UI hosts:** `ConsoleApp1`, `WinFormsApp1`, and `WpfApp1` are runnable hosts; `Mvvm.Console`, `Mvvm.Winforms`, and `Mvvm.Wpf` are MVVM infrastructure per UI framework, each referencing `Mvvm` (for example, `Mvvm.Console` references `Mvvm` and uses `System.ComponentModel.Composition` for plugin-style assembly loading).
- **Feature + tests:** `UserManagement` references `BusinessLayer`, `DataAccessLayer.Hibernate`, `Logging`, and `Mvvm`; `TestProject1` is the test project — `BusinessLayer` declares `InternalsVisibleTo("TestProject1")`, so the tests reach into `BusinessLayer` internals.

### Relationships

- Depends on the 16 `.csproj` files it lists; the `Path` values are relative to the repository root, where this file lives.
- It does not express project dependencies itself — inter-project references live in each `.csproj` (verified, e.g., `UserManagement` → `BusinessLayer`/`Mvvm`, `BusinessLayer` → `DataAccessLayer.Hibernate`/`Dto`/`Logging`). The solution file only says *which* projects belong to the solution.
- Its "caller" is the build tooling: `dotnet build Demo.slnx`, `dotnet test Demo.slnx`, or simply opening the file in an IDE.

### Behavior & Details

- Each `<Project>` element has a `Path` (required) and an optional `Id`. Most entries carry an `Id` GUID, but `BusinessLayer`, `Dto`, and `Entity.Hibernate` do not — the format derives one for those, which is still valid.
- The file contains no solution folders, no solution-level build configuration, and no platform/conditional project entries. Everything in the solution is a plain project reference.

### Rough edges

The project names are a mix of meaningful names (`BusinessLayer`, `UserManagement`, `DataAccessLayer`) and default template names (`ConsoleApp1`, `WinFormsApp1`, `WpfApp1`, `TestProject1`), so the role of the three host apps and the test project has to be inferred from their contents rather than their names.