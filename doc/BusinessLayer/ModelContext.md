# BusinessLayer/ModelContext.cs

### Purpose

`ModelContext` is the MEF export that supplies the application's database connection settings: the connection string and the list of assembly names that contain ORM mapping files. It exists because the business layer needs a concrete, discoverable `IModelContext` that consumers can import without knowing where the configuration comes from.

The whole class is a single parameterless constructor — the type has no members of its own.

### Role in the Codebase

It is a small DI/MEF composition piece inside `BusinessLayer`. The `DataAccessLayer.Hibernate` project defines the contract (`IModelContext`), the abstract base (`ModelContextBase`), and the ORM machinery; this file is the concrete binding for this application, filling in the actual connection string and mapping assembly.

### Relationships

- **Base class:** `ModelContextBase` in `DataAccessLayer.Hibernate/ModelContextBase.cs`. That abstract class stores `ConnectionString` and `MappingAssemblies` as read-only properties, so this class inherits both.
- **Exported contract:** `IModelContext` (defined in `DataAccessLayer.Hibernate/IModelContext.cs`). The `[Export(typeof(IModelContext))]` attribute makes MEF provide this class to any part that imports `IModelContext`.
- **Caller:** `RepositoryFactory` in `BusinessLayer/RepositoryFactory.cs` imports `IModelContext` in its `[ImportingConstructor]` and passes `context.ConnectionString` and `context.MappingAssemblies` on to its `HibernateRepositoryFactory<int>` base. That is the only current consumer found in the codebase.
- **Configuration:** The connection string is read from `ConfigurationManager.AppSettings["ConnectionString"]`, so the value is whatever `app.config`/`web.config` supplies under that key.

### Behavior & Details

The single parameterless constructor does two things:

- Reads `AppSettings["ConnectionString"]` and passes it (with the `!` null-forgiving operator) as the `connectionString` argument to `ModelContextBase`. If the setting is missing, the constructor receives `null`; nothing here validates it, so a misconfigured app will fail later, when a connection is actually attempted.
- Passes `["Entity.Hibernate"]` as the `mappingAssemblies` argument, telling the Hibernate ORM that entity mapping files live in the `Entity.Hibernate` assembly. Note that `ModelContextBase`'s own commented-out line adds `"DataAccessLayer"` and `"DataAccessLayer.Hibernate"` to this list in some deployments — the base class keeps its list as passed, so this export maps only `Entity.Hibernate`.

Because the constructor takes no parameters, MEF can compose it unconditionally; all its configuration comes from `AppSettings` at construction time. Changing the connection string or mapping assemblies is done entirely through configuration and this one call, respectively — no code change in the data-access layer is needed.

### Rough edges

- `ConfigurationManager.AppSettings["ConnectionString"]!` asserts non-null without checking; a missing app setting is a silent `null` until the connection string is used.
- The `["Entity.Hibernate"]` literal hard-codes the mapping assembly for this app; adding a mapping assembly requires editing this file rather than the configuration.