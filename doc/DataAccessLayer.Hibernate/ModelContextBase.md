### Purpose

This file defines `ModelContextBase`, the abstract base class that packages the two pieces of configuration NHibernate needs to build a session factory — a SQL Server connection string and the list of assemblies containing Fluent NHibernate mapping files — and exposes them through `IModelContext`. It exists so the composition root (the `BusinessLayer` project) can supply this configuration while the `DataAccessLayer.Hibernate` project consumes it without knowing where it comes from.

### Role in the Codebase

It lives in `DataAccessLayer.Hibernate/`, alongside `IModelContext` (`IModelContext.cs`), `HibernateRepositoryFactory`, `SessionFactoryBuilder`, and the repository/unit-of-work types. It is the reference implementation of the `IModelContext` contract: a thin state container with two read-only properties. Nothing in the data-access project consumes `ModelContextBase` directly — consumers take the `IModelContext` interface, and the concrete `ModelContext` (in `BusinessLayer/ModelContext.cs`) derives from this class and is `[Export]`-ed via MEF. Removing it would break `ModelContext`, and with it the construction of `RepositoryFactory`, which reads `context.ConnectionString` and `context.MappingAssemblies`.

### Relationships

- **Implements `IModelContext`** (defined in `IModelContext.cs`, same directory). The interface declares exactly the two read-only properties — `ConnectionString` and `MappingAssemblies` — so this class is a straightforward implementation with no default behavior to override.
- **Derived by `BusinessLayer.ModelContext`** (`BusinessLayer/ModelContext.cs`), the only concrete subclass. That class passes `ConfigurationManager.AppSettings["ConnectionString"]` and `["Entity.Hibernate"]` into this base constructor and marks the instance as an MEF export of `IModelContext`.
- **Consumed via the interface** by `BusinessLayer.RepositoryFactory`, whose constructor takes an `IModelContext` and forwards `context.ConnectionString` / `context.MappingAssemblies` to `HibernateRepositoryFactory<TKey>`, which builds the NHibernate session factory.

### Behavior & Details

- The constructor is `protected`, enforcing derivation: it stores the `connectionString` in `ConnectionString` and the `mappingAssemblies` array in `MappingAssemblies`. Both properties are get-only, so a context's configuration is fixed for its lifetime.
- `MappingAssemblies` is built as a `List<string>` copy of the input array with an empty placeholder slot where `"DataAccessLayer"` and `"DataAccessLayer.Hibernate"` are commented out — presumably a leftover from a time when those assemblies carried mappings. As written, the stored array is just a copy of whatever the caller passed (e.g. `["Entity.Hibernate"]`).
- The values themselves are only validated when `HibernateRepositoryFactory` or `SessionFactoryBuilder` actually load the named assemblies via `Assembly.Load(mappingAssembly)` and connect using the string — this class does no validation of its own.

### Rough edges

- The commented-out entries in the `List<string>` initializer are dead weight with no behavioral effect; the array ends up identical to a plain copy of the input.
- A missing `AppSettings["ConnectionString"]` entry reaches this constructor as `null` (the `!` in `ModelContext` only silences the compiler), so an absent configuration surfaces later as an NHibernate failure rather than a clear error here.