# WinFormsApp1/App.config

### Purpose

Runtime configuration for the `WinFormsApp1` executable. It supplies a single `appSettings` entry, `ConnectionString`, that points at a local SQL Server instance: `Server=localhost;Database=Users;Integrated Security=True;TrustServerCertificate=True;`. Nothing else in the project is configurable.

### Role in the Codebase

This is the project's sole source of database configuration. The WinForms code itself (`Program.cs`, `Controller/`, `View/`) never reads it directly; the value is consumed transitively through the shared `BusinessLayer`. At build time MSBuild copies it alongside the output as `WinFormsApp1.exe.config`, and at runtime .NET's configuration system (`System.Configuration.ConfigurationManager`) resolves `AppSettings["ConnectionString"]` against it. Removing or renaming the key would not fail the build — the app would only start failing later, when the data layer first attempts to open a database connection.

### Relationships

- **`BusinessLayer/ModelContext.cs`** — the direct consumer. Its primary constructor argument is `ConfigurationManager.AppSettings["ConnectionString"]!`, passed (with a null-forgiving operator) to `ModelContextBase`. This class exports `IModelContext` via MEF.
- **`DataAccessLayer.Hibernate/ModelContextBase.cs`** — stores the string in the read-only `ConnectionString` property.
- **`BusinessLayer/RepositoryFactory.cs`** — imports `IModelContext` through its `[ImportingConstructor]` and forwards `context.ConnectionString` into `DataAccessLayer.Hibernate/HibernateRepositoryFactory.cs`, which configures NHibernate's `SessionFactoryBuilder` with it.

The same one-key file is duplicated in `ConsoleApp1/App.config` and `WpfApp1/App.config`; the `WinFormsApp1` copy exists because each host executable carries its own `.exe.config`.

### Behavior & Details

- The value is an MS SQL connection string using Windows integrated authentication against a `Users` database on `localhost`. `TrustServerCertificate=True` suppresses the TLS certificate check.
- The key name is exactly `ConnectionString`; the lookup in `ModelContext` uses that literal string, so renaming the key breaks the lookup silently rather than at compile time.
- There is no fallback if the key is absent: the `!` in `ModelContext` hides the null until NHibernate tries to build a session factory, at which point the failure surfaces as a data-access error deep in the MEF construction chain, not as a configuration error.
- The `TrustServerCertificate` setting is a per-machine deployment concern baked into source control; changing it requires editing the file and rebuilding.