I have enough context to document this file accurately.

# RepositoryFactory.cs

**Path:** `BusinessLayer/RepositoryFactory.cs`

### Purpose

`RepositoryFactory` exists to make business-layer services able to obtain NHibernate-backed repositories without knowing anything about how the database session is created. It is the composition root for the persistence stack: it wires a connection string and the mapping assembly from `IModelContext` into the NHibernate infrastructure, and exposes repository instances to the rest of the business layer through the `IRepositoryFactory` contract.

### Role in the Codebase

This class is a thin, MEF-activated glue class in the `BusinessLayer` project. It does no business logic of its own — it closes over the generic, infrastructure-level `HibernateRepositoryFactory<TKey>` (defined in `DataAccessLayer.Hibernate/HibernateRepositoryFactory.cs`) by pinning `TKey` to `int`, and adds the `IRepositoryFactory` identity so that the layer only depends on its own interface.

It is exported to the MEF container via `[Export(typeof(IRepositoryFactory))]` and constructed through its `[ImportingConstructor]`, so it is never instantiated directly in code. Services like `UserService` (`BusinessLayer/UserService.cs`) take `IRepositoryFactory` in their constructor and call `Create()` to get a disposable repository per operation. If this class were removed, the container would have no export satisfying `IRepositoryFactory`, and every business service that depends on it would fail to compose.

### Relationships

- **Base class:** `HibernateRepositoryFactory<int>` — a concrete generic class in `DataAccessLayer.Hibernate` whose `protected` constructor takes a connection string and an array of mapping-assembly names and builds an NHibernate `ISessionFactory` using FluentNHibernate (SQL Server, with a schema update). `RepositoryFactory` inherits its entire lifecycle from there and adds no members of its own beyond the constructor.
- **Implemented interface:** `IRepositoryFactory` (`BusinessLayer/IRepositoryFactory.cs`), which is an empty marker interface over `IHibernateRepositoryFactory<int>` (`DataAccessLayer.Hibernate/IHibernateRepositoryFactory.cs`). That interface in turn extends `IRepositoryFactory<IHibernateRepository<TKey>, TKey>`, so `Create()` returns a stateless-session-backed `IHibernateRepository<int>`.
- **Import:** `IModelContext` (`DataAccessLayer.Hibernate/IModelContext.cs`), resolved at composition time to the layer's `ModelContext` (`BusinessLayer/ModelContext.cs`), which supplies the `ConnectionString` from app config and `MappingAssemblies` as `["Entity.Hibernate"]`.
- **Callers:** MEF-composed business services that import `IRepositoryFactory` — currently `UserService`.

### Behavior & Details

- The constructor forwards `context.ConnectionString` and `context.MappingAssemblies` to the base NHibernate factory, which does all real work: FluentNHibernate configuration for a SQL Server 2012 provider, a `SchemaUpdate` on startup, loading fluent mappings from the named assemblies, and building the session factory.
- Because the base constructor runs on construction, the NHibernate session factory is built exactly when MEF composes `RepositoryFactory`. The base class also guards against a second construction with a static `IsInitialized` flag and throws `InvalidOperationException` ("Already initialized") if a second instance is ever requested, and wraps any setup failure in a `DataAccessException`.
- `Create()` (inherited) returns a new `IRepository<int>` per call over a freshly opened stateless session; callers dispose the repository after use (as `UserService` does with `using var repository = ...`).
- Note that `HibernateRepositoryFactory.Drop()` is a `NotImplementedException` stub, so there is no working teardown path for the session factory in the current code.

### Rough edges

- `ModelContext` is defined as `public class ModelContext()` — a primary constructor taking no parameters that are actually used; the arguments (`ConnectionString`, `MappingAssemblies`) come from the `ModelContextBase` base-class constructor instead. This compiles in C# 12, but it looks like a leftover and is worth confirming is intentional.
- The one-instance-per-process enforcement in the base class relies on a static flag, so a second composition (e.g., a second container or a test re-composing the graph) throws rather than sharing or rebuilding.
- `Drop()` is unimplemented, so the session factory lifecycle has no symmetric teardown to its construction.