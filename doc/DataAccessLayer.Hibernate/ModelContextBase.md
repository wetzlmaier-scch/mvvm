# `DataAccessLayer.Hibernate/ModelContextBase.cs`

### Purpose

This file defines `ModelContextBase`, an abstract base class that packages the two pieces of configuration NHibernate needs to build a session factory — the database `ConnectionString` and the list of `MappingAssemblies` to scan for Fluent NHibernate mappings. It exists so that the concrete `ModelContext` in `BusinessLayer` can be a one-line class: everything it does is already done by its base.

### Role in the Codebase

The class is the single reference implementation of `IModelContext` (declared in `DataAccessLayer.Hibernate/IModelContext.cs`, the two-property interface in the same directory). `IModelContext` is the seam between the composition root (`BusinessLayer`) and the ORM machinery in this project: the composition root produces an instance, and the data-access layer consumes it as a contract, never as this concrete type.

If removed, `BusinessLayer/ModelContext.cs` would have nothing to derive from, and the MEF export of `IModelContext` would fail to compose `RepositoryFactory`. Nothing in `DataAccessLayer.Hibernate` itself references `ModelContextBase` by name — only the `IModelContext` interface is what this project's types accept.

### Relationships

- **Implements `IModelContext`** (`IModelContext.cs`, same directory): the interface declares exactly the two get-only properties `ConnectionString` and `MappingAssemblies`, so `ModelContextBase` satisfies it with no additional members.
- **Subclassed by `BusinessLayer.ModelContext`** (`BusinessLayer/ModelContext.cs`): a primary-constructor class that calls the base constructor with `ConfigurationManager.AppSettings["ConnectionString"]!` and the hard-coded literal `["Entity.Hibernate"]` (the assembly holding the `User` entity and its Fluent NHibernate mapping). It is exported to MEF as `IModelContext`.
- **Consumed indirectly** by `BusinessLayer.RepositoryFactory` (`BusinessLayer/RepositoryFactory.cs`), whose `[ImportingConstructor]` takes an `IModelContext` and forwards `context.ConnectionString` and `context.MappingAssemblies` to `HibernateRepositoryFactory`'s protected constructor.

### Behavior & Details

- The constructor is `protected`, which enforces derivation: `ModelContextBase` cannot be instantiated directly, and both properties are get-only, so the values are fixed for the lifetime of any `IModelContext` instance.
- The constructor stores `connectionString` into `ConnectionString` unchanged, but `mappingAssemblies` is copied into a `List<string>` before being stored via `MappingAssemblies = list.ToArray()`. The `List<string>` construction wraps an empty object initializer — `/*"DataAccessLayer", "DataAccessLayer.Hibernate" */` is a commented-out entry that would have added the layer assemblies themselves to the scan list. As written, no assemblies are appended; the stored array is just a defensive copy of the input.
- Downstream, `HibernateRepositoryFactory<TKey>` calls `Assembly.Load(assembly)` for each name in `MappingAssemblies` and scans the loaded assembly with `FluentMappings.AddFromAssembly(...)`. So the entries here are **assembly names**, not namespaces — a typo or a renamed assembly surfaces only when the session factory is built, wrapped in a `DataAccessException`.

### Rough edges

- The commented-out `"DataAccessLayer", "DataAccessLayer.Hibernate"` entries are unexplained: it is not possible to tell from the code whether mappings once lived in these assemblies or the comment is a leftover, but their absence means only the caller-supplied assemblies are scanned.
- The `List<string>` wrapper serves no purpose other than hosting that commented-out initializer — it is dead machinery around a copy.