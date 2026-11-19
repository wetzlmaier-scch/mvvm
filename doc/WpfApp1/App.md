# `WpfApp1/App.config`

### Purpose

Runtime configuration for the `WpfApp1` executable. It holds a single `appSettings` entry, `ConnectionString`, pointing at a local SQL Server instance: `Server=localhost;Database=Users;Integrated Security=True;TrustServerCertificate=True`. That is the entire file — no other sections, no connectionStrings block, no binding redirects.

It exists so the WPF app has a database connection string to load at startup. Nothing in the WPF project itself reads it; the consuming code lives in the shared business layer.

### Role in the Codebase

The WPF project contains `App.xaml`/`App.xaml.cs` (startup), `View/`, `Controller/`, and this file. It is the only place this project supplies a database connection. At build time the .NET SDK copies it to `WpfApp1.exe.config`, and `System.Configuration` makes it visible to everything running in that process — including the `BusinessLayer` and `DataAccessLayer.Hibernate` assemblies the app references.

If it were removed (or the key renamed), startup would still succeed, but the first NHibernate operation would fail: `ModelContext` reads the setting with a null-forgiving operator, so the `null` surfaces later as a `NullReferenceException` rather than a clear configuration error.

### Relationships

- **Producer of the setting:** this file.
- **Direct consumer:** `BusinessLayer/ModelContext.cs` — its primary constructor argument is `ConfigurationManager.AppSettings["ConnectionString"]!`, passed to `ModelContextBase` (in `DataAccessLayer.Hibernate/ModelContextBase.cs`), which stores it as the read-only `ConnectionString` property.
- **Downstream flow:** `ModelContext` is MEF-exported as `IModelContext`; `BusinessLayer/RepositoryFactory.cs` imports it and forwards `context.ConnectionString` into the NHibernate `SessionFactoryBuilder`. The WPF app's view models reach this chain without ever touching `App.config` directly.
- **Sibling copies:** `ConsoleApp1/App.config` and `WinFormsApp1/App.config` hold the same entry, since each host executable needs its own `.exe.config`. Changes here must be replicated there if the other hosts should point at the same database.

### Behavior & Details

- The key name is exactly `ConnectionString`. Code looks it up by that literal string, so renaming the key breaks the lookup with no compile-time warning.
- `Integrated Security=True` means Windows authentication; no credentials are stored in the file.
- `TrustServerCertificate=True` suppresses SQL Server certificate validation — appropriate for a `localhost` dev database, but not something to copy into a production connection string.

### Rough edges

- The file is a development-only default hardcoded to `localhost`. Every host in the repo ships the identical value, so the database location can only change by editing the config file and rebuilding — there is no environment-based override.