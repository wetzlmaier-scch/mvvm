# `BusinessLayer/ModelContext.cs`

### Purpose

`ModelContext` is the concrete, MEF-exported `IModelContext` for this application. It exists to hand the NHibernate data-access layer the two configuration values it needs — the database connection string and the names of the assemblies that contain Fluent NHibernate mappings. All of the actual configuration storage is inherited; this class only supplies the values and makes itself discoverable.

### Role in the Codebase

It is a wiring piece at the edge of `BusinessLayer`, not business logic. `BusinessLayer` is the layer that composes the application's MEF graph (its `RepositoryFactory` and `UserService` are also `[Export]`ed), and `ModelContext` is the part of that graph where configuration enters from the host process. It has no behaviour of its own: if it were removed, nothing in `BusinessLayer` would stop compiling, but `RepositoryFactory`'s `[ImportingConstructor]` would no longer be satisfiable and the container would fail to build a repository factory at startup.

### Relationships

- **Base class:** `ModelContextBase` in `DataAccessLayer.Hibernate/ModelContextBase.cs`, an abstract class in the data-access project that implements `IModelContext`. It stores the constructor arguments in the read-only `ConnectionString` and `MappingAssemblies` properties. `ModelContext` inherits both properties and overrides nothing.
- **Exported contract:** `IModelContext` in `DataAccessLayer.Hibernate/IModelContext.cs`, a two-property interface (`ConnectionString`, `MappingAssemblies`). The `[Export(typeof(IModelContext))]` attribute (from `System.ComponentModel.Composition`, .NET's MEF) is what ties this class to that contract.
- **Consumer:** `RepositoryFactory` in `BusinessLayer/RepositoryFactory.cs` declares `[ImportingConstructor] public RepositoryFactory(IModelContext context)` and forwards `context.ConnectionString` and `context.MappingAssemblies` to its base `HibernateRepositoryFactory<int>`, which builds the NHibernate session factory.
- **Composition host:** the hosts (`WinFormsApp1/Program.cs`, `WpfApp1`, `ConsoleApp1/Program.cs`) build a `CompositionContainer` via `CompositionHelper` in `Common/Composition/CompositionHelper.cs`, which is where this export gets discovered and matched to the import.

### Behavior & Details

The class uses a C# 12 primary constructor — the parentheses after the type name are empty because every value comes from the base-constructor invocation:

```csharp
public class ModelContext()
    : ModelContextBase(ConfigurationManager.AppSettings["ConnectionString"]!, ["Entity.Hibernate"]);
```

Two values flow into `ModelContextBase` at construction:

- **Connection string** comes from `ConfigurationManager.AppSettings["ConnectionString"]` — the `appSettings` entry in the host's `App.config` (each host carries one, e.g. `WinFormsApp1/App.config` points at a local `Users` database). The null-forgiving operator `!` asserts the setting exists; there is no fallback or error if the key is missing.
- **Mapping assemblies** is the hard-coded literal `["Entity.Hibernate"]`, the project that holds the `User` entity and its Fluent NHibernate mapping. At session-factory build time, `HibernateRepositoryFactory` loads each name via `Assembly.Load`, so the entry must be the assembly name, not a path. Note that `ModelContextBase`'s constructor has a commented-out append of `"DataAccessLayer", "DataAccessLayer.Hibernate"` to this list; those assemblies are therefore *not* scanned for mappings as the code stands.

Because it has no members, there is no other state, control flow, or failure mode here — construction either supplies the two values or throws if the app setting is absent.

### Rough edges

- The connection string is read unconditionally in the primary constructor with no guard, so constructing a `ModelContext` in a context without a host `App.config` (e.g. a unit test) throws a `NullReferenceException` rather than a meaningful configuration error.
- The mapping assembly list is hard-coded to `["Entity.Hibernate"]`, so adding a new entity/mapping assembly requires a code change here rather than a configuration change.
- The class's parameterless primary constructor `ModelContext()` is never used by anything; the only real construction path is the MEF import satisfied by `RepositoryFactory`.