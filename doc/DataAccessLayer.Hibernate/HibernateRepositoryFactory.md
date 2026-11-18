# `DataAccessLayer.Hibernate/HibernateRepositoryFactory.cs`

### Purpose

This file defines `HibernateRepositoryFactory<TKey>`, the class that builds the NHibernate `ISessionFactory` once per process and then mints repository instances from it. It exists so that business-layer code can ask for `IHibernateRepository<TKey>` instances without knowing how NHibernate is configured, which assemblies contain the Fluent mappings, or how the schema is kept current.

### Role in the Codebase

`HibernateRepositoryFactory<TKey>` is the bootstrap point of the `DataAccessLayer.Hibernate` project: it is the only place in the wired-up code path that actually talks to FluentNHibernate and opens sessions. The concrete class is not used directly — `BusinessLayer.RepositoryFactory` (`BusinessLayer/RepositoryFactory.cs`) derives from `HibernateRepositoryFactory<int>` and is MEF-exported as `IRepositoryFactory`, so every repository in the application flows through this class's constructor and `Create()`.

If it were removed, the business layer would have nothing to resolve `IRepositoryFactory` from, and no session factory would ever be built (the sibling `SessionFactoryBuilder` duplicates some of this configuration logic but is not called anywhere in the repository).

### Relationships

- **Implements** `IHibernateRepositoryFactory<TKey>` (same folder), which itself derives from `DataAccessLayer.IRepositoryFactory<IHibernateRepository<TKey>, TKey>` (`DataAccessLayer/IRepositoryFactory.cs`) and declares `new IHibernateRepository<TKey> Create()`.
- **Produces** `HibernateRepository<TKey>` (same folder) — one instance per `Create()` call, each wrapping its own `IStatelessSession` opened from `_sessionFactory`.
- **Subclassed by** `BusinessLayer.RepositoryFactory`, whose MEF importing constructor forwards `IModelContext.ConnectionString` and `IModelContext.MappingAssemblies` into this class's protected constructor. Because the constructor is `protected`, `HibernateRepositoryFactory<TKey>` can only be instantiated through a subclass — the only one that exists is `RepositoryFactory`.
- Uses FluentNHibernate and NHibernate (third-party) to configure a SQL Server 2012 database with the Microsoft data provider and to load Fluent mappings from the named assemblies.

### Behavior & Details

**Constructor** — `protected HibernateRepositoryFactory(string connectionString, string[] mappingAssemblies)`:

1. Guards on a **static** `IsInitialized` flag. Because it is static and lives on the generic type, the guard is per `TKey`: only the *first* constructed `HibernateRepositoryFactory<SomeTKey>` in the process succeeds; any second construction of the same closed type throws `InvalidOperationException("Already initialized")`. It does not prevent constructing `HibernateRepositoryFactory<int>` and `HibernateRepositoryFactory<string>` side by side — only one factory per key type.
2. Builds a FluentNHibernate configuration: `MsSql2012` dialect, the given connection string, `MicrosoftDataSqlClientDriver`, and `ShowSql()` (SQL statements are printed to the console, always).
3. In `ExposeConfiguration`, runs `new SchemaUpdate(cfg).Execute(false, true)` — so constructing the factory issues `ALTER TABLE`/`CREATE` DDL against the live database as a side effect of startup.
4. For each name in `mappingAssemblies`, calls `Assembly.Load(assembly)` and adds that assembly's Fluent mappings to the configuration. Note the inputs are assembly *names as strings*, not `Assembly` objects.
5. Calls `BuildSessionFactory()` and stores the result in `_sessionFactory`. `IsInitialized` is set to `true` only after this succeeds.
6. Any exception in steps 2–5 is caught and rethrown wrapped in a `DataAccessException` with the message "Error initializing Hibernate repository factory".

**`Create()`** — Opens a new `IStatelessSession` from the stored factory and returns a fresh `HibernateRepository<TKey>` around it. Each call gives the caller an independent, disposable repository; there is no pooling or caching of sessions. (Stateless sessions mean no identity map and no first-level cache, which is why the `Save` path in `HibernateRepository` uses `InsertAsync`.)

**Explicit interface implementation** — `IRepositoryFactory<IHibernateRepository<TKey>, TKey>.Create()` is implemented explicitly and simply delegates to the public `Create()`. This satisfies the base `DataAccessLayer.IRepositoryFactory<TRepository, TKey>.Create()` slot without creating a second public overload.

**`Drop()`** — Declared in `DataAccessLayer.IRepositoryFactory`, but here it throws `NotImplementedException`. There is no working teardown path for the session factory anywhere in the repository; callers do not use it.

### Rough edges

- `Drop()` is an unimplemented stub, so the session factory is never disposed or reconfigured; the only lifecycle is build-once, run-forever.
- The `IsInitialized` flag is static on the generic type yet only ever set in the success path: if a second construction is attempted for the same `TKey` while the first constructor is mid-flight, the check can race. In practice this does not matter because MEF constructs `RepositoryFactory` once, but the flag gives no protection at all across different `TKey` values and does not survive a failed startup followed by a retry of a *different* key type ordering.
- Schema DDL (`SchemaUpdate`) runs inside the constructor with `ShowSql()` always on, so any construction attempt mutates the database and logs SQL — even one that ends in a thrown `DataAccessException` (the DDL in `ExposeConfiguration` runs before `BuildSessionFactory`).