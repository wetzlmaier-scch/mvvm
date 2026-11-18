# `SessionFactoryBuilder.cs`

**Path:** `DataAccessLayer.Hibernate/SessionFactoryBuilder.cs`

### Purpose

`SessionFactoryBuilder` is a static helper that builds an NHibernate `ISessionFactory` from a connection string and a list of mapping-assembly names. Its intent is to centralize the Fluent NHibernate configuration in one place. In practice it is currently dead code: `BuildSessionFactory` is not called anywhere in the repository, and the same configuration logic is duplicated inline in the constructor of `HibernateRepositoryFactory<TKey>`.

### Role in the Codebase

It sits in the `DataAccessLayer.Hibernate` project alongside `HibernateRepository<TKey>`, `HibernateRepositoryFactory<TKey>`, `HibernateUnitOfWork`, and the `IModelContext` / `ModelContextBase` types. The class that actually does bootstrapping in the running application is `HibernateRepositoryFactory<TKey>` (in `HibernateRepositoryFactory.cs`), whose constructor is invoked from `RepositoryFactory` in the `BusinessLayer`. `SessionFactoryBuilder` is a parallel, unused copy of that bootstrapping.

### Relationships

- **NHibernate / FluentNHibernate / `Microsoft.Data.SqlClient`** (external): the ORM and ADO.NET driver. `Fluently.Configure()` from FluentNHibernate drives the whole configuration; `MicrosoftDataSqlClientDriver` is NHibernate's driver wrapper around the `Microsoft.Data.SqlClient` assembly.
- **`IModelContext`** (`DataAccessLayer.Hibernate/IModelContext.cs`) / **`ModelContextBase`** (`DataAccessLayer.Hibernate/ModelContextBase.cs`): the abstract base class exposes `ConnectionString` and `MappingAssemblies`, exactly the two inputs `BuildSessionFactory` takes. `SessionFactoryBuilder` is not wired to that base class today; `HibernateRepositoryFactory` is.

The intended caller would be the same as `HibernateRepositoryFactory`'s — a composition root that holds an `IModelContext` and needs an `ISessionFactory`. No such caller exists in the repository right now.

### Behavior & Details

`BuildSessionFactory(string connectionString, string[] mappingAssemblies)`:

1. Builds the database config: `MsSqlConfiguration.MsSql2012` with the given connection string, the `MicrosoftDataSqlClientDriver`, and `ShowSql()` enabled (SQL is logged to the console).
2. For each name in `mappingAssemblies`, loads the assembly with `Assembly.Load(name)` and registers every Fluent NHibernate mapping in it via `FluentMappings.AddFromAssembly`.
3. Runs `SchemaUpdate.Execute(false, true)` — NHibernate's auto-update of the schema based on the mappings. `false` means do not perform a drop-and-recreate on startup; `true` means log the DDL.
4. Returns the result of `.BuildSessionFactory()`.

Differences worth knowing versus the sibling `HibernateRepositoryFactory<TKey>` constructor:

- **Order of operations:** here `SchemaUpdate` runs *before* mappings are added to the configuration, whereas in `HibernateRepositoryFactory` it runs *after*. The `SchemaUpdate(cfg)` here is invoked via `ExposeConfiguration` on the `.Database(...)` chain, so the mappings loop happens on the already-schema-updated configuration. Whether the two orders produce the same DDL in practice is worth verifying before deleting either copy.
- **No error wrapping.** Exceptions from `Assembly.Load` or `BuildSessionFactory` propagate raw to the caller. By contrast, `HibernateRepositoryFactory<TKey>` wraps initialization failures in a `DataAccessException` and guards against a second init with an `IsInitialized` static flag.
- **`ShowSql()`** is on in both copies; if you keep `SessionFactoryBuilder` as the canonical path, decide what logging level production should have.

### Rough edges

The entire Fluent NHibernate configuration is duplicated between `SessionFactoryBuilder.BuildSessionFactory` and the `HibernateRepositoryFactory<TKey>` constructor, and only the factory's copy is active. Anyone editing one will not see the other, so the two will drift.