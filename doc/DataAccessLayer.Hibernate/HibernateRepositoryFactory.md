# `DataAccessLayer.Hibernate/HibernateRepositoryFactory.cs`

### Purpose

This file defines `HibernateRepositoryFactory<TKey>`, the class that builds the NHibernate `ISessionFactory` once per process and then mints repository instances from it. It exists so that business-layer code can get working `IHibernateRepository<TKey>` instances without ever touching NHibernate configuration, drivers, or mapping assemblies.

### Role in the Codebase

It is the bootstrap point of the `DataAccessLayer.Hibernate` project: the only place in the wired-up code path that talks to FluentNHibernate and opens NHibernate sessions. Its constructor is `protected` by design — the class is meant to be subclassed. `BusinessLayer/RepositoryFactory.cs` closes it at `TKey = int` and forwards the connection string and mapping-assembly names from an injected `IModelContext`.

If it were removed, `BusinessLayer.RepositoryFactory` (MEF-exported as `IRepositoryFactory`) would have nothing to inherit from, and no session factory would ever be built for the application. The sibling `SessionFactoryBuilder.cs` duplicates most of this configuration logic but is not called by the wired-up path.

### Relationships

- **Implements** `IHibernateRepositoryFactory<TKey>` (same folder), which itself derives from `IRepositoryFactory<IHibernateRepository<TKey>, TKey>` in `DataAccessLayer/IRepositoryFactory.cs`. The base generic interface declares `TRepository Create()` and `void Drop()`.
- **Subclassed by** `BusinessLayer.RepositoryFactory`, whose MEF importing constructor forwards `context.ConnectionString` and `context.MappingAssemblies` into this class's protected constructor.
- **Creates** `HibernateRepository<TKey>` (same folder), constructed with a stateless NHibernate session.
- **Downstream consumers** reach it through the business-layer `IRepositoryFactory` interface — for example `UserService` (`BusinessLayer/UserService.cs`) calls `_repositoryFactory.Create()` inside a `using` block for each operation.

### Behavior & Details

**Constructor** — `protected HibernateRepositoryFactory(string connectionString, string[] mappingAssemblies)`:

1. Guards on a **static** `IsInitialized` flag. Because it is static and lives on the generic type, the guard is process-global per `TKey`: a second `HibernateRepositoryFactory<int>` (or a new MEF activation of `RepositoryFactory`) throws `InvalidOperationException("Already initialized")`. In practice MEF's default sharing policy yields one instance, so this path is mostly defensive.
2. Builds the FluentNHibernate configuration: `MsSql2012` dialect with the given connection string, `MicrosoftDataSqlClientDriver`, and `ShowSql()` enabled (SQL is always echoed, even in production).
3. Runs `SchemaUpdate(cfg).Execute(false, true)` against the database at startup — the schema is auto-updated from the Fluent mappings, and it runs even when the connection is read-only.
4. Loads each named assembly via `Assembly.Load(assembly)` and scans it for Fluent mappings. A bad assembly name or connection string surfaces here, wrapped in `DataAccessException("Error initializing Hibernate repository factory", e)`.
5. Builds the `ISessionFactory` and flips `IsInitialized` to `true`.

**`Create()`** — Returns `new HibernateRepository<TKey>(_sessionFactory.OpenStatelessSession())`. Each call opens a **stateless** session, so repositories get no first-level cache and `Save` on the repository performs an `Insert` (an entity that already exists in the table will fail at the database level, not become an update). The caller is expected to dispose the repository, which closes the session.

**Explicit interface implementation** — `IRepositoryFactory<IHibernateRepository<TKey>, TKey>.Create()` is implemented explicitly and simply delegates to the public `Create()`. This satisfies the base `DataAccessLayer.IRepositoryFactory<TRepository, TKey>` contract; a caller holding the non-Hibernate interface gets the same repository instance.

**`Drop()`** — Declared in `DataAccessLayer.IRepositoryFactory` but here it throws `NotImplementedException`. There is no working teardown path for the session factory anywhere in the repository; any caller invoking `Drop()` at shutdown will crash.

### Rough edges

- The static `IsInitialized` guard is scoped per `TKey` specialization, so `HibernateRepositoryFactory<int>` and `HibernateRepositoryFactory<Guid>` would both be constructable in the same process — the flag does not actually guarantee "one session factory per process" across key types.
- NHibernate configuration (dialect, driver, schema update, mapping loading) is duplicated nearly verbatim in `SessionFactoryBuilder.cs`, which no wired-up code path calls.
- `SchemaUpdate` executing on every startup means the schema is a by-product of the mappings, with no migration control.
- `Drop()` is part of the public contract (`DataAccessLayer.IRepositoryFactory`) but is unimplemented.