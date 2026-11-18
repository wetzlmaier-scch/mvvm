# `DataAccessLayer.Hibernate/IModelContext.cs`

### Purpose

`IModelContext` is a two-property contract that carries everything the data-access layer needs to configure NHibernate: the database `ConnectionString` and the `MappingAssemblies` (assembly names containing Fluent NHibernate mappings). It exists so that `DataAccessLayer.Hibernate` can build its NHibernate session factory without knowing where the connection settings come from — the composition root in another project supplies them through this interface.

### Role in the Codebase

The interface is the configuration seam between `BusinessLayer` (composition root) and `DataAccessLayer.Hibernate` (ORM machinery). It lives in the data-access project because that is the side that *consumes* the values; the business layer only provides an implementation. If it were removed, `ModelContextBase`, `BusinessLayer.ModelContext`, and `RepositoryFactory` would all have no shared contract to work against — the connection string and mapping list would have no typed channel between the layers.

### Relationships

- **Implemented by** `ModelContextBase` (same directory, `ModelContextBase.cs`): the abstract base in this project that stores both properties, read via get-only auto-properties set in its constructor. Concrete contexts derive from it rather than from the interface directly.
- **Concrete implementation:** `ModelContext` in `BusinessLayer/ModelContext.cs`, a primary-constructor class over `ModelContextBase` that reads the connection string from `ConfigurationManager.AppSettings["ConnectionString"]` and hard-codes `["Entity.Hibernate"]` as the mapping assemblies. It is MEF-exported with `[Export(typeof(IModelContext))]`.
- **Consumed by** `RepositoryFactory` in `BusinessLayer/RepositoryFactory.cs`, whose `[ImportingConstructor]` takes an `IModelContext` and forwards `context.ConnectionString` and `context.MappingAssemblies` to the `HibernateRepositoryFactory<int>` base constructor. That base constructor uses the two values to configure NHibernate (MS SQL 2012 driver, schema update) and load each named assembly's Fluent mappings.

### Behavior & Details

The interface declares only two read-only members:

- `string[] MappingAssemblies` — the names of the assemblies to load and scan for Fluent NHibernate mappings. `HibernateRepositoryFactory`'s constructor calls `Assembly.Load(assembly)` for each name, so entries must be loadable assembly names (like `"Entity.Hibernate"`), not namespaces or file paths.
- `string ConnectionString` — the raw connection string, interpreted by the consumer as an MS SQL 2012 connection (the driver and dialect are fixed in `HibernateRepositoryFactory`, not configurable through this interface).

Both properties are getters only; the values are fixed for the lifetime of the context instance, which is constructed once by the MEF container and passed through to the single `RepositoryFactory` instance (which itself is single-use — its base constructor throws `InvalidOperationException("Already initialized")` on a second construction).