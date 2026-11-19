# `DataAccessLayer.Hibernate/IModelContext.cs`

### Purpose

This file defines `IModelContext`, the contract for supplying the two configuration values the NHibernate data-access layer needs to bootstrap: a database `ConnectionString` and the list of `MappingAssemblies` (assembly names containing Fluent NHibernate mapping definitions). It exists so that `DataAccessLayer.Hibernate` can build its session factory without knowing *where* the configuration comes from — that detail (e.g. reading `App.config`) is pushed to whichever project supplies the concrete implementation.

### Role in the Codebase

This is one of the seam interfaces in the `DataAccessLayer.Hibernate` project, alongside `IHibernateRepository`, `IHibernateRepositoryFactory`, and `IHibernateUnitOfWork`. Its role is to let the business layer depend on an abstracted configuration provider instead of the ORM directly. If this interface were removed, the NHibernate factory code would have no decoupled way to receive its configuration, and the MEF wiring between the business layer's `ModelContext` and `RepositoryFactory` would break.

### Relationships

- **Implemented by** `ModelContextBase` in `DataAccessLayer.Hibernate/ModelContextBase.cs`, an abstract base class in the same project. It stores constructor arguments and exposes them as the two read-only properties.
- **Concrete implementation:** `ModelContext` in `BusinessLayer/ModelContext.cs`. This class extends `ModelContextBase` and is MEF-exported with `[Export(typeof(IModelContext))]`. It reads `ConfigurationManager.AppSettings["ConnectionString"]` and `["Entity.Hibernate"]` and passes them to the base class.
- **Consumed by** `RepositoryFactory` in `BusinessLayer/RepositoryFactory.cs`. Its `[ImportingConstructor]` takes an `IModelContext` and forwards `context.ConnectionString` and `context.MappingAssemblies` to the `HibernateRepositoryFactory<TKey>` base constructor.

The flow: MEF injects the exported `ModelContext` into `RepositoryFactory`, which then hands the values through to `HibernateRepositoryFactory<int>`.

### Behavior & Details

The interface has two read-only properties:

- **`ConnectionString`** — The ADO.NET connection string used by Fluent NHibernate's `MsSql2012` configuration (SQL Server driver). The value is ultimately passed to `Fluently.Configure().Database(...)`.
- **`MappingAssemblies`** — An array of assembly *names* (strings) that the factory loads via `Assembly.Load` to discover Fluent NHibernate mappings. `HibernateRepositoryFactory` iterates this array and calls `configuration.Mappings(m => m.FluentMappings.AddFromAssembly(Assembly.Load(assembly)))` for each entry.

Because both properties are get-only, implementors must capture the values at construction time (or via backing fields); they cannot be lazily re-evaluated. The contract is intentionally minimal: it carries exactly what the session factory needs and nothing more.