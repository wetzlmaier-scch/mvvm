# `SessionFactoryBuilder.cs`

**Path:** `DataAccessLayer.Hibernate/SessionFactoryBuilder.cs`

## Purpose

`SessionFactoryBuilder` is a static helper that builds an NHibernate `ISessionFactory` from a connection string and a list of mapping-assembly names. It exists to centralise the Fluent NHibernate configuration (SQL Server 2012, the SqlClient driver, Fluent mapping discovery, and schema auto-update) in one place, as a reusable alternative to the inline configuration in `HibernateRepositoryFactory<TKey>`.

## Role in the Codebase

It sits in the `DataAccessLayer.Hibernate` project alongside `HibernateRepository<TKey>`, `HibernateRepositoryFactory<TKey>`, `HibernateUnitOfWork`, and the `IModelContext`/`ModelContextBase` types. It is intended as a shared factory for the session infrastructure that repositories (`HibernateRepository<TKey>`) and the unit of work depend on.

In practice, it is currently dead code: `BuildSessionFactory` is not called anywhere in the repository. The same configuration is instead duplicated inline in the constructor of `HibernateRepositoryFactory<TKey>` (via the `BusinessLayer/ModelContext.cs` → `BusinessLayer/RepositoryFactory.cs` chain, where `IModelContext.ConnectionString` and `MappingAssemblies` are passed to the factory's constructor). If you want new code to use this class, nothing stops you from calling `SessionFactoryBuilder.BuildSessionFactory(...)` directly — it is public. If you remove the file, nothing breaks today, but the duplicated configuration in `HibernateRepositoryFactory` remains the only working path.

## Relationships

- Depends on external libraries only: Fluent NHibernate (`Fluently.Configure()`, `MsSqlConfiguration.MsSql2012`), NHibernate (`ISessionFactory`, `MicrosoftDataSqlClientDriver`), and `NHibernate.Tool.hbm2ddl.SchemaUpdate`. No other project file is referenced.
- `mappingAssemblies` are resolved at runtime with `Assembly.Load(mappingAssembly)`, so the entries are assembly names, not file paths. The live counterpart in this codebase passes `["Entity.Hibernate"]` (see `BusinessLayer/ModelContext.cs`), so the mapping assemblies contain Fluent NHibernate maps such as `UserMap`.
- Its output, an `ISessionFactory`, is the same type `HibernateRepositoryFactory<TKey>` holds and uses via `_sessionFactory.OpenStatelessSession()` to create `HibernateRepository<TKey>` instances.

## Behavior & Details

`BuildSessionFactory(string connectionString, string[] mappingAssemblies)`:

1. Configures the database with `MsSqlConfiguration.MsSql2012`, the supplied `connectionString`, `MicrosoftDataSqlClientDriver`, and `ShowSql()` (SQL statements are logged to the console/stdout).
2. For each name in `mappingAssemblies`, loads the assembly and adds its Fluent mappings via `FluentMappings.AddFromAssembly`.
3. Runs `SchemaUpdate(cfg).Execute(false, true)` through `ExposeConfiguration`. This auto-updates the database schema to match the mappings on every build — the `true` flag means updates are executed, `false` means no script is written to disk.
4. Returns the result of `.BuildSessionFactory()`.

Caveats:

- **Schema mutation at construction time.** `SchemaUpdate` runs as a side effect of building the factory, so calling this method in a production or test environment will issue `ALTER` statements against the target database. The duplicated inline version in `HibernateRepositoryFactory<TKey>` has the same behaviour, so the whole project assumes an auto-evolving schema.
- **No error wrapping.** Exceptions from `Assembly.Load` or `BuildSessionFactory` propagate raw to the caller. By contrast, `HibernateRepositoryFactory<TKey>` wraps initialisation failures in a `DataAccessException`.
- **`ShowSql()` is hardcoded on.** There is no way to suppress SQL logging through this method.

## Rough edges

- The entire Fluent NHibernate configuration is duplicated between `SessionFactoryBuilder.BuildSessionFactory` and the `HibernateRepositoryFactory<TKey>` constructor, and only the factory's copy is actually used by the application. The two could diverge (they already differ in ordering: `SessionFactoryBuilder` registers mappings before the `SchemaUpdate` step, the factory registers them after) with no compile-time signal.