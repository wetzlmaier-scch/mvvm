# `DataAccessLayer.Hibernate/IModelContext.cs`

### Purpose

This file defines `IModelContext`, a small two-property contract that carries the database connection string and the list of mapping assemblies needed to configure NHibernate. It exists so that consumers in the data-access layer can receive this configuration through dependency injection rather than reading `App.config` or hard-coding assembly names themselves.

### Role in the Codebase

`IModelContext` is the configuration seam between the composition root (the `BusinessLayer` project) and the `DataAccessLayer.Hibernate` project. The data-access layer knows how to build an NHibernate `ISessionFactory` given a connection string and mapping assemblies, but it does not know *where* those values come from. The `BusinessLayer` project supplies them: `BusinessLayer/ModelContext.cs` exports a concrete `ModelContext` (built on `ModelContextBase`) with `[Export(typeof(IModelContext))]`, pulling the connection string from `ConfigurationManager.AppSettings["ConnectionString"]` and the mapping assemblies from a hard-coded array.

### Relationships

- **`ModelContextBase`** (`DataAccessLayer.Hibernate/ModelContextBase.cs`): the abstract base class in the same project that implements `IModelContext`. Concrete contexts (like `BusinessLayer.ModelContext`) inherit from it, which stores the constructor-supplied values in the two read-only properties.
- **`BusinessLayer/RepositoryFactory.cs`**: the main consumer. Its `[ImportingConstructor]` takes an `IModelContext` and forwards `context.ConnectionString` and `context.MappingAssemblies` to `HibernateRepositoryFactory<TKey>`'s base constructor.
- **`BusinessLayer/ModelContext.cs`**: the concrete implementation that MEF exports for this interface.
- **`HibernateRepositoryFactory<TKey>`** (`DataAccessLayer.Hibernate/HibernateRepositoryFactory.cs`): receives the two values and uses them with FluentNHibernate to configure the database, load Fluent mappings from the named assemblies, run a schema update, and build the `ISessionFactory`.

### Behavior & Details

- **`MappingAssemblies`** (`string[]`): the names of the assemblies that contain FluentNHibernate mapping files. `SessionFactoryBuilder` and `HibernateRepositoryFactory` each iterate over this array, call `Assembly.Load(name)`, and register the mappings found in each assembly. A caller adding a new entity-mapping assembly therefore only needs to update the concrete context, not the data-access layer.
- **`ConnectionString`** (`string`): passed to `MsSqlConfiguration.MsSql2012.ConnectionString(...)` along with `MicrosoftDataSqlClientDriver`, i.e. this interface assumes a SQL Server database.
- Both members are read-only getters; the interface is implemented once in `ModelContextBase`, which assigns the values in its constructor. There is no default implementation, no state, and no lifetime management — it is a pure data contract.

Rough edges: `SessionFactoryBuilder.BuildSessionFactory` duplicates the entire session-factory configuration logic of `HibernateRepositoryFactory`'s constructor (including the `SchemaUpdate` call), so `SessionFactoryBuilder` and `HibernateRepositoryFactory` can drift apart when NHibernate configuration changes. Also, `HibernateRepositoryFactory` has a static `IsInitialized` flag that throws `InvalidOperationException("Already initialized")` if a second factory is ever constructed, meaning the injected `IModelContext` is effectively usable by exactly one factory instance per process.