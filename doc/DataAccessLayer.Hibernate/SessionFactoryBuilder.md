### Purpose

**`DataAccessLayer.Hibernate/SessionFactoryBuilder.cs`**

`SessionFactoryBuilder` is a static helper whose job is to build an NHibernate `ISessionFactory` from a connection string and a list of mapping-assembly names. It exists to centralize the Fluent NHibernate bootstrap: pick a SQL Server dialect, load the Fluent mappings from each given assembly, auto-update the database schema to match those mappings, and then construct the (expensive) session factory that everything else in the data layer uses.

The single public method is `BuildSessionFactory(string connectionString, string[] mappingAssemblies)`, which returns the `ISessionFactory`.

### Role in the Codebase

It lives in the `DataAccessLayer.Hibernate` project next to `HibernateRepository<TKey>`, `HibernateRepositoryFactory<TKey>`, `HibernateUnitOfWork`, and the `IModelContext` / `ModelContextBase` types.

It is effectively a second, parallel implementation of the same bootstrap that `HibernateRepositoryFactory<TKey>` performs in its constructor — same dialect (`MsSql2012`), same driver (`MicrosoftDataSqlClientDriver`), same `ShowSql()`, same `SchemaUpdate` execution, same `Assembly.Load` per mapping assembly. The only callers of the repository's session-factory logic are the `HibernateRepositoryFactory<TKey>` path; `SessionFactoryBuilder.BuildSessionFactory` is not called anywhere else in the repository. Its `mappingAssemblies` parameter takes *names* of assemblies rather than `Assembly` instances, which fits the shape of the values stored by `IModelContext` / `ModelContextBase` (a connection string plus a list of assembly names), so the intended caller is whatever code reads those values and wants a ready-made factory without going through the repository factory.

### Relationships

- **Fluent NHibernate / NHibernate** (external libraries): `Fluently.Configure()`, `MsSqlConfiguration.MsSql2012`, `MicrosoftDataSqlClientDriver`, `ISessionFactory`, and `SchemaUpdate` all come from NHibernate and Fluent NHibernate.
- **`IModelContext` / `ModelContextBase`** (same project): the likely source of the two inputs — a `ConnectionString` and an array of mapping-assembly names.
- **`Entity.Hibernate`-style mapping assemblies** (other projects): the assemblies passed in are scanned for `ClassMap<T>` subclasses via `FluentMappings.AddFromAssembly`, so the actual entity mappings live in those assemblies, not here.
- **`HibernateRepositoryFactory<TKey>`** (same project): duplicates this exact configuration; only the factory's copy is active in the running flow.

### Behavior & Details

- The dialect is hard-wired to `MsSqlConfiguration.MsSql2012` with the SQL Server `MicrosoftDataSqlClientDriver`, so this only targets SQL Server.
- `ShowSql()` is always on, meaning SQL statements are logged whenever a factory is built through this path.
- The `foreach` loops over `mappingAssemblies` and calls `Assembly.Load(mappingAssembly)` on each name before adding that assembly's Fluent mappings. A misspelled or unloadable assembly name throws an `AssemblyLoadException` from `Assembly.Load` with no wrapping or custom message.
- `ExposeConfiguration(cfg => new SchemaUpdate(cfg).Execute(false, true))` runs hbm2ddl's schema update *before* `BuildSessionFactory()`, so the database schema is altered to match the mappings each time a factory is built. The `true` flag also logs the generated SQL.
- `BuildSessionFactory()` is the expensive step — it compiles all mappings into the configuration — and its result is returned directly. There is no caching, disposal handling, or initialization guard; the caller owns the returned factory.

### Rough edges

- The entire Fluent NHibernate configuration is duplicated between `SessionFactoryBuilder.BuildSessionFactory` and the `HibernateRepositoryFactory<TKey>` constructor, and only the factory's copy is active, so changes to the dialect, driver, or schema-update strategy must be made in two places (or one copy will drift).
- `BuildSessionFactory` has no callers in this repository, so its behavior — including schema auto-update and `ShowSql()` — is not exercised by anything here.
- `Assembly.Load(mappingAssembly)` takes an assembly *name*; it will throw `FileNotFoundException`/`AssemblyLoadException` for anything that isn't exactly resolvable by that name.