### Purpose

`ConsoleApp1/App.config` (path: `ConsoleApp1/App.config`) is the application configuration file for the `ConsoleApp1` host executable. It exists to supply the SQL Server connection string that the composition-based data layer needs at runtime.

### Role in the Codebase

The file holds a single `appSettings` entry, `ConnectionString`, pointing at a local SQL Server instance: `Server=localhost;Database=Users;Integrated Security=True;TrustServerCertificate=True;`. It is the runtime counterpart of the `BusinessLayer` and `DataAccessLayer.Hibernate` projects: `ModelContext` in `BusinessLayer/ModelContext.cs` reads this exact key at construction time via `ConfigurationManager.AppSettings["ConnectionString"]` and hands it to `ModelContextBase`, which uses it to build the Hibernate session factory (`SessionFactoryBuilder`). Without this file — or without the `ConnectionString` key — `ModelContext` receives `null` (the code uses the null-forgiving operator `!` rather than checking) and the session factory build fails, so no part of the user-management demo in `ConsoleApp1/Program.cs` can run.

The same key and value are duplicated in `WinFormsApp1/App.config` and `WpfApp1/App.config`, which means each host application carries its own copy of the connection string rather than sharing one configuration source.

### Relationships

- **Consumer:** `BusinessLayer/ModelContext.cs` is the only reader of this key. It resolves the app settings through `System.Configuration.ConfigurationManager`, which picks up the entry from the active executable's `App.config`.
- **Downstream chain:** `ModelContextBase` (`DataAccessLayer.Hibernate/ModelContextBase.cs`) stores the string in its `ConnectionString` property; `HibernateRepositoryFactory` and `SessionFactoryBuilder` in the same project apply it to Hibernate's configuration.
- **Host:** `ConsoleApp1/Program.cs` builds a `CompositionContainer` via `CompositionHelper.CreateCompositionContainer(["ConsoleApp1.exe"])` (from `Common/Composition/CompositionHelper.cs`), which discovers exported parts from the loaded assemblies — including `ModelContext` — and triggers the config read. The console app itself never reads the config file directly.

### Behavior & Details

- The connection uses Windows integrated security (`Integrated Security=True`) against a `localhost` server and a database named `Users`; `TrustServerCertificate=True` skips TLS certificate validation, typical of a local development setup.
- The value is a plain literal with no `&lt;connectionStrings&gt;` section, environment-variable substitution, or per-environment overrides. Changing the database location means editing this key in every host's `App.config` that shares it.
- `Program.cs` passes `"ConsoleApp1.exe"` to the composition helper, but that helper's string overload treats the argument as a name of an assembly to *ignore* (its comment in `Program.cs` notes the value is "not a valid .net assembly"), so this call ignores nothing in practice; it does not affect which config file is read.