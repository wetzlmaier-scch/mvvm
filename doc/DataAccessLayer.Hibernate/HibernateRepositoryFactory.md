# `DataAccessLayer.Hibernate/HibernateRepositoryFactory.cs`

### Purpose

This file defines `HibernateRepositoryFactory<TKey>`, the class that builds the NHibernate `ISessionFactory` once for the application and then mints repository instances from it. It exists so that repository creation can be decoupled from the costly one-time NHibernate bootstrap (database configuration, schema update, mapping assembly loading) — the bootstrap happens in the constructor, and every subsequent repository is just a new stateless session.

It implements `IHibernateRepositoryFactory<TKey>` (same folder), which in turn derives from `DataAccessLayer.IRepositoryFactory<IHibernateRepository<TKey>, TKey>` defined in `DataAccessLayer/IRepositoryFactory.cs`. That base interface declares `TRepository Create()` and `void Drop()`; this class adds the concrete `new IHibernateRepository<TKey> Create()` signature from the derived interface.

### Role in the Codebase

Inside `DataAccessLayer.Hibernate`, this class is the entry point to the data layer. The business layer never touches NHibernate directly; it gets repositories through the factory chain:

- `BusinessLayer.RepositoryFactory` (in `BusinessLayer/RepositoryFactory.cs`) inherits `HibernateRepositoryFactory<int>` and implements `IRepositoryFactory` (i.e. `IHibernateRepositoryFactory<int>`), passing `context.ConnectionString` and `context.MappingAssemblies` from an `IModelContext` into the base constructor. It is exported via MEF (`[Export(typeof(IRepositoryFactory))]`).
- Consumers such as `BusinessLayer.UserService` depend on `IRepositoryFactory` and call `_repositoryFactory.Create()` inside a `using` block per operation.

If this class were removed, `BusinessLayer.RepositoryFactory` would have no base class to bootstrap from, and no repository could be created. Note the factory itself is created only once in the process — a second construction attempt is an error (see below).

### Relationships

- **`IHibernateRepositoryFactory<TKey>`** (`DataAccessLayer.Hibernate/IHibernateRepositoryFactory.cs`) — the interface implemented here.
- **`HibernateRepository<TKey>`** (`DataAccessLayer.Hibernate/HibernateRepository.cs`) — what `Create()` returns; it wraps the `IStatelessSession` this factory opens, exposing `GetQueryable`, `Find`, `Save`, a lazily created `UnitOfWork` (`HibernateUnitOfWork`), and `Dispose()`.
- **`IModelContext`** (`IModelContext.cs` / `ModelContextBase.cs`) — the source of the `connectionString` and `mappingAssemblies` arguments, supplied by the caller (`RepositoryFactory`) rather than read from configuration here.
- **FluentNHibernate / NHibernate** (external libraries) — `Fluently.Configure()`, `MsSqlConfiguration`, `MicrosoftDataSqlClientDriver`, `SchemaUpdate` are all from those packages.

### Behavior & Details

**Constructor** `HibernateRepositoryFactory(string connectionString, string[] mappingAssemblies)`:

- **Static `IsInitialized` guard.** A second call to any constructor throws `InvalidOperationException("Already initialized")`. The flag is `static`, so this is process-wide: the application is designed to have exactly one session factory.
- **Database configuration** is hardcoded to SQL Server 2012 via `MsSqlConfiguration.MsSql2012`, using the `MicrosoftDataSqlClientDriver` and `.ShowSql()` (SQL statements echoed to the log).
- **Schema auto-update:** `ExposeConfiguration(cfg => new SchemaUpdate(cfg).Execute(false, true))` runs an NHibernate schema *update* against the database on every factory construction, before the session factory is built. This mutates the database schema to match the mappings.
- **Mappings:** for each assembly name in `mappingAssemblies`, `Assembly.Load(assembly)` loads it by string name and all its Fluent NHibernate mappings are added. The assembly names are whatever `IModelContext` provides (e.g. `"DataAccessLayer.Hibernate"`).
- **Failure:** any exception during this setup is wrapped in `DataAccessException("Error initializing Hibernate repository factory", e)`. Note the static flag is set only after success, so a failed construction allows a retry — but a successful one permanently blocks further construction for the process lifetime.

**`Create()`** returns `new HibernateRepository<TKey>(_sessionFactory.OpenStatelessSession())`. Each repository gets its **own stateless session** — stateless sessions hold no identity map or first-level cache, so every repository instance is independent and must be disposed by the caller (the business layer does this with `using var repository = ...`). The factory holds only the `_sessionFactory`; repositories are not tracked or pooled.

**Explicit interface implementation.** `IRepositoryFactory<IHibernateRepository<TKey>, TKey>.Create()` is implemented explicitly and simply delegates to `Create()`, satisfying the base `DataAccessLayer.IRepositoryFactory<TRepository, TKey>` contract through the interface hierarchy.

**`Drop()`** throws `NotImplementedException` — the schema-teardown half of the interface contract is a stub; there is no way to drop the schema through this factory.

### Rough edges

- The session factory construction logic here (connection string, `SchemaUpdate`, loading mappings by assembly name) duplicates `SessionFactoryBuilder.BuildSessionFactory` in the same folder, except the factory's version also runs the schema update. The two can drift.
- `Drop()` is unimplemented, and `Create()` returns a stateless session with no disposal tracking — callers must remember to dispose each repository, and there is nothing in the factory to catch them if they don't.