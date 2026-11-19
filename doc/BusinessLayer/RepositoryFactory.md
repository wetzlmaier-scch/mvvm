# `BusinessLayer/RepositoryFactory.cs`

### Purpose

This file defines `RepositoryFactory`, the business layer's concrete, MEF-exported factory for NHibernate repositories. It exists to make a data-access dependency injectable into business services: it takes its database configuration (`IModelContext`), passes it down to the generic `HibernateRepositoryFactory<int>` in the data-access layer, and publishes itself as the `IRepositoryFactory` contract that services consume. It contains no logic of its own — it is a glue point between MEF composition and the ORM bootstrap.

### Role in the Codebase

It is the only concrete implementation of `BusinessLayer.IRepositoryFactory` in the repository, so if it were removed the composition containers in the host applications (`ConsoleApp1`, `WinFormsApp1`, `Mvvm.Wpf`) could not resolve `IRepositoryFactory`, and any service importing it (currently `UserService`) would fail to construct.

It is one half of a small pair: `ModelContext` (`BusinessLayer/ModelContext.cs`) exports the `IModelContext` this class imports, and together the two are what connect the application's configuration to NHibernate.

### Relationships

- **Base class:** `HibernateRepositoryFactory<TKey>` in `DataAccessLayer.Hibernate/HibernateRepositoryFactory.cs`. This class closes it at `TKey = int`, the key type of the application's entities. All real work — building the `ISessionFactory`, applying schema updates, creating repositories — is inherited from there.
- **Implemented interface:** `IRepositoryFactory` from `BusinessLayer/IRepositoryFactory.cs`, an empty non-generic interface that extends `IHibernateRepositoryFactory<int>`. The `[Export(typeof(IRepositoryFactory))]` attribute publishes the class under that contract.
- **Injected dependency:** `IModelContext` from `DataAccessLayer.Hibernate/IModelContext.cs` (a two-property contract: `ConnectionString`, `MappingAssemblies`). The export satisfying this import is `ModelContext` in the same directory.
- **Consumers:** `UserService` (`BusinessLayer/UserService.cs`) takes `IRepositoryFactory` in its `[ImportingConstructor]` and calls `Create()` inside a `using` block per operation.

### Behavior & Details

The single constructor is the entire behavioural surface of this class:

```csharp
[ImportingConstructor]
public RepositoryFactory(IModelContext context) : base(context.ConnectionString, context.MappingAssemblies)
```

MEF instantiates it with an `IModelContext` (in practice `ModelContext`, which supplies `ConfigurationManager.AppSettings["ConnectionString"]` and the hardcoded assembly name `Entity.Hibernate`), and this constructor forwards both values into the base `HibernateRepositoryFactory` protected constructor. That base constructor is where everything happens: it configures FluentNHibernate for SQL Server 2012, runs a `SchemaUpdate`, loads Fluent mappings from each named assembly, and builds the `ISessionFactory`.

Three consequences worth knowing before touching this class:

- **It must be constructed exactly once per process.** The base class guards against a second construction with a `static` `IsInitialized` flag and throws `InvalidOperationException("Already initialized")`. MEF's per-container singleton semantics normally satisfy this; building a second `CompositionContainer` in the same process would not.
- **Failures surface at construction, not use.** A bad connection string or unloadable mapping assembly is wrapped by the base class in a `DataAccessException("Error initializing Hibernate repository factory", e)` and thrown during composition, i.e. as soon as `UserService` (or any importer) is first resolved.
- **`Create()` and `Drop()` come from the base class.** `Create()` opens a stateless session and returns a `HibernateRepository<int>`; `Drop()` is part of the inherited `IRepositoryFactory<TRepository, TKey>` contract but throws `NotImplementedException` in the base — there is no working teardown path for the session factory in this repository.

### Rough edges

- Because the base constructor throws if initialized, this factory cannot be constructed twice in one process. Combined with the `NotImplementedException` in `Drop()`, there is no supported way to rebuild the factory (for example, after a configuration change) without restarting the process.