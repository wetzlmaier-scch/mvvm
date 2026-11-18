# `DataAccessLayer.Hibernate/ModelContextBase.cs`

### Purpose

This file defines `ModelContextBase`, an abstract base class that packages the two pieces of configuration NHibernate needs to build a session factory: a SQL Server connection string and a list of assembly names that contain the fluent mapping files. It is the shared half of the data-access configuration for the NHibernate layer.

### Role in the Codebase

`ModelContextBase` is a thin base class in the `DataAccessLayer.Hibernate` namespace. It implements the `IModelContext` interface (defined in `IModelContext.cs`, same directory), which exposes exactly two properties: `ConnectionString` and `MappingAssemblies`. The class stores whatever is passed to its constructor and exposes it read-only — it performs no validation, no I/O, and no database work.

Its job is to give the BusinessLayer a common contract for "here is where and how to talk to the database" without the business layer having to know about NHibernate's internals.

### Relationships

- **Implements `IModelContext`** (same directory, `IModelContext.cs`) — the interface declares the same two read-only properties, so `ModelContextBase` is essentially the reference implementation.
- **Extended by `ModelContext`** in `BusinessLayer/ModelContext.cs`. That class is a single-line concrete subclass:

  ```csharp
  public class ModelContext()
      : ModelContextBase(ConfigurationManager.AppSettings["ConnectionString"]!, ["Entity.Hibernate"]);
  ```

  It pulls the connection string from `App.config`/`Web.config` and points the mapping assemblies at `Entity.Hibernate`. It is MEF-exported as `IModelContext`.
- **Consumed indirectly by `RepositoryFactory`** in `BusinessLayer/RepositoryFactory.cs`, which takes an `IModelContext` in its importing constructor and forwards `context.ConnectionString` and `context.MappingAssemblies` to its base, `HibernateRepositoryFactory<int>`. That factory then builds an `ISessionFactory` using those values.
- Note the sibling static helper `SessionFactoryBuilder` in the same directory does essentially the same thing with the same two inputs, but it is **not** the one wired up by the business layer — `HibernateRepositoryFactory` contains its own inline FluentNHibernate configuration.

### Behavior & Details

- The constructor is `protected`, so the class can only be instantiated through subclasses. It copies the connection string as-is and turns the mapping-assemblies array into a (temporarily growable) `List<string>` before freezing it back into the `MappingAssemblies` array. Nothing is verified: an empty list or a misspelled assembly name will surface later, as an `Assembly.Load` failure, inside `HibernateRepositoryFactory`'s constructor.
- Both properties are get-only, set once at construction, so a context instance is effectively immutable once created.

### Rough edges

- The `List<string>` in the constructor is only used to carry a commented-out initializer `{ /*"DataAccessLayer", "DataAccessLayer.Hibernate" */}`. The list adds no behavior — `list.ToArray()` on an empty list just returns the original array. If the commented assemblies are ever re-enabled, only the comment would need to change.
- `SessionFactoryBuilder` in the same directory duplicates the NHibernate bootstrapping that `HibernateRepositoryFactory` already does inline, with the same inputs. Nothing in the business layer appears to call `SessionFactoryBuilder`, so it is currently dead parallel code, and the connection-string/mapping-assemblies pair exists in two places that can drift.