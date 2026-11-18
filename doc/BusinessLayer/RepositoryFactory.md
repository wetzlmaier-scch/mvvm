# `BusinessLayer/RepositoryFactory.cs`

### Purpose

`RepositoryFactory` is the MEF-exported, NHibernate-backed repository factory that business-layer services use to open repositories against the database. It exists so that services like `UserService` can ask for repositories without knowing anything about NHibernate, session factories, connection strings, or mapping assemblies. All of that plumbing lives in `DataAccessLayer.Hibernate`; this class only pins down *how this application* configures that plumbing.

It is the BusinessLayer's concrete point where composition meets data access. It closes the generic `HibernateRepositoryFactory<TKey>` at `TKey = int` (the application's entity key type) and publishes the result under the BusinessLayer's own `IRepositoryFactory` contract.

### Role in the Codebase

This class is a thin, MEF-activated glue class. It does no business logic — it has no methods of its own and no fields. Everything is inherited from `HibernateRepositoryFactory<int>` (defined in `DataAccessLayer.Hibernate/HibernateRepositoryFactory.cs`) or obtained through its `[ImportingConstructor]`.

It is part of the BusinessLayer's composition set. The host applications (`ConsoleApp1`, `WinFormsApp1`, `Mvvm.Wpf`) build a `CompositionContainer` via `Common/Composition/CompositionHelper.cs`, which scans assemblies for `[Export]`-annotated types. `RepositoryFactory` is one such type. If it were removed, `UserService` — which imports `IRepositoryFactory` in its `[ImportingConstructor]` — would fail to compose at startup, and the entire BusinessLayer would stop resolving.

### Relationships

- **Base class:** `HibernateRepositoryFactory<TKey>` from `DataAccessLayer.Hibernate/HibernateRepositoryFactory.cs`. This is the generic, engine-specific factory. `RepositoryFactory` specializes it to `HibernateRepositoryFactory<int>` and adds no members. Everything the class does (session-factory construction, `Create()`, `Drop()`) is inherited.
- **Implemented interface:** `IRepositoryFactory` from `BusinessLayer/IRepositoryFactory.cs`. That interface is a non-generic marker that simply extends `IHibernateRepositoryFactory<int>` (from `DataAccessLayer.Hibernate/IHibernateRepositoryFactory.cs`), so implementing it is satisfied by the inherited `Create()` and `Drop()`.
- **Injected dependency:** `IModelContext` from `DataAccessLayer.Hibernate/IModelContext.cs`, imported through the `[ImportingConstructor]`. In practice the export satisfying that import is `BusinessLayer/ModelContext.cs` (`ModelContext`), which pulls `ConnectionString` from `AppSettings["ConnectionString"]` and supplies `["Entity.Hibernate"]` as the mapping assembly.
- **Consumers:** `UserService` (`BusinessLayer/UserService.cs`) imports `IRepositoryFactory` and calls `Create()` inside a `using` block for each operation. Any future service in the BusinessLayer would follow the same pattern.

### Behavior & Details

The class has one constructor, marked `[ImportingConstructor]` for MEF:

```csharp
[ImportingConstructor]
public RepositoryFactory(IModelContext context)
    : base(context.ConnectionString, context.MappingAssemblies)
{ }
```

It does exactly one thing: forwards the connection string and mapping assemblies from the injected `IModelContext` to the base class. The base constructor (`HibernateRepositoryFactory<TKey>`) is where all the real work happens — it calls `Fluently.Configure()` to build a SQL Server 2012 session factory, runs a `SchemaUpdate` against the configuration, loads fluent mappings from each named assembly, and caches the result in a static `IsInitialized` flag.

A few inherited behaviors are worth knowing because they surface through this class:

- **Singleton-by-throw.** The base constructor uses a `static bool IsInitialized` and throws `InvalidOperationException("Already initialized")` on the second call. Because MEF's default sharing policy is `Any`, this is fine — the container instantiates exactly one `RepositoryFactory`. If anything ever constructs `new RepositoryFactory(context)` directly by hand, the first such call succeeds and every subsequent one throws.
- **`Create()` opens a stateless session.** Each call returns a new `HibernateRepository<TKey>` bound to a fresh `OpenStatelessSession()`. Callers must dispose the repository to release the session.
- **`Drop()` is not implemented.** The inherited `Drop()` throws `NotImplementedException`, so do not use it for shutdown logic.
- **Configuration is fixed to SQL Server.** The base hard-codes `MsSqlConfiguration.MsSql2012` and `MicrosoftDataSqlClientDriver`; the only knobs this class (via `ModelContext`) supplies are the connection string and the mapping-assembly names.
- **Failure surfaces at construction time.** If the connection string is bad or a mapping assembly cannot be loaded, the base constructor wraps the original exception in a `DataAccessException("Error initializing Hibernate repository factory", e)` and throws. The first attempt to resolve `IRepositoryFactory` from the container is where you will see that.

### Rough edges

- `ModelContext.cs` is declared as `public class ModelContext()` — a primary constructor with no parameters — yet it derives from `ModelContextBase(...)` using arguments that look like constructor-initializer syntax. As written it is not valid C#; the file likely has a typo or is mid-refactor. `RepositoryFactory` itself is not at fault, but its MEF import of `IModelContext` depends on `ModelContext` compiling and exporting correctly.
- The static `IsInitialized` guard in `HibernateRepositoryFactory<TKey>` means "already initialized" is a process-global state shared across every `TKey` specialization, not per-instance. In this repository only `int` is used, so it is a latent hazard rather than a live one.