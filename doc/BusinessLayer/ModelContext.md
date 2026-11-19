### Purpose

`BusinessLayer/ModelContext.cs` defines `ModelContext`, the concrete `IModelContext` for this application. It exists to hand the NHibernate data-access layer the two configuration values it needs to build a session factory: the database connection string and the list of assemblies containing entity mappings. Without it, MEF could not construct the repository factory.

The entire class is one declaration: a MEF export attribute, a primary constructor, and a base-class initializer. All behaviour lives in the base class.

### Role in the Codebase

`ModelContext` is a wiring piece at the edge of `BusinessLayer`, not business logic. `BusinessLayer` is the MEF composition root of the app: it exports `ModelContext` and the `RepositoryFactory` that consumes it. Nothing else in `BusinessLayer` or the host applications (`ConsoleApp1`, `WinFormsApp1`, `WpfApp1`) references `ModelContext` directly; the only real construction path is the MEF import satisfied for `RepositoryFactory`.

If this class were removed, the MEF graph would have no export of `IModelContext`, and `RepositoryFactory`'s `[ImportingConstructor]` could not be satisfied, so no repository would be created.

### Relationships

- **Base class:** `ModelContextBase` in `DataAccessLayer.Hibernate/ModelContextBase.cs`, an abstract class in the data-access project that implements `IModelContext`. It takes the connection string and mapping-assembly list in its protected constructor and exposes them as read-only properties. All the state of this class is actually stored there.
- **Exported contract:** `IModelContext` in `DataAccessLayer.Hibernate/IModelContext.cs`, a two-property interface (`ConnectionString`, `MappingAssemblies`). The `[Export(typeof(IModelContext))]` attribute makes this class satisfy imports of that interface elsewhere in the MEF catalog.
- **Consumer:** `RepositoryFactory` in `BusinessLayer/RepositoryFactory.cs` declares `[ImportingConstructor] public RepositoryFactory(IModelContext context)` and forwards `context.ConnectionString` and `context.MappingAssemblies` into its base class, `HibernateRepositoryFactory<int>`.
- **Configuration source:** the connection string comes from `ConfigurationManager.AppSettings["ConnectionString"]`, set in each host app's `App.config` (e.g. `ConsoleApp1/App.config`) as a SQL Server connection string.
- **Mapping assembly:** the literal `"Entity.Hibernate"` refers to the sibling `Entity.Hibernate` project, which holds the `User` entity and its Fluent NHibernate mapping. `BusinessLayer` references that project in its `.csproj`.

### Behavior & Details

Two values flow into `ModelContextBase` at construction:

- **Connection string:** read from application settings via `ConfigurationManager.AppSettings["ConnectionString"]!`. The null-forgiving operator suppresses the nullable warning but does not guard against a missing key — see "Rough edges".
- **Mapping assemblies:** the hard-coded array `["Entity.Hibernate"]`. At session-factory build time, `HibernateRepositoryFactory` (in `DataAccessLayer.Hibernate`) loads mappings from these assemblies. If an entity mapping is added to a different assembly, this literal is the place to update.

Note that `ModelContextBase`'s constructor builds its stored array from the passed-in list plus an (empty) additional set — a commented-out `/*"DataAccessLayer", "DataAccessLayer.Hibernate"*/` suggests more mapping assemblies were contemplated. As written, only `"Entity.Hibernate"` is scanned.

Construction has no other logic, no lifecycle, and no side effects beyond capturing the two values. The class has no members of its own.

### Rough edges

- The connection string is read unconditionally in the primary constructor with no guard, so constructing a `ModelContext` without a host `App.config` key (e.g. in a unit test) throws a `NullReferenceException` far from the missing configuration.
- The parameterless primary constructor `ModelContext()` is never used directly; the only real construction path is the MEF import. The primary-constructor syntax here is doing the work of an ordinary base-class initializer and may confuse readers expecting a base constructor call to be a statement.