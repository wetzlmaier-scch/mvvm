### Purpose

`App.config` is the runtime configuration file for the `WpfApp1` project. It holds a single `appSettings` entry, `ConnectionString`, pointing at a local SQL Server instance: `Server=localhost;Database=Users;Integrated Security=True;TrustServerCertificate=True;`. At build time it is transformed into `WpfApp1.exe.config`, which the .NET configuration system reads at startup via `ConfigurationManager.AppSettings`.

### Role in the Codebase

It is the only place this project supplies a database connection string. The WPF application itself (`App.xaml`, `App.xaml.cs`, the `View/` and `Controller/` directories) does not read the config directly; it reaches the value indirectly through the shared `BusinessLayer` project. Removing this file (or the key) means `ConfigurationManager.AppSettings["ConnectionString"]` returns `null`, and — because the consumer uses the null-forgiving operator — the failure surfaces later, when NHibernate actually tries to open a connection, rather than at startup with a clear error.

### Relationships

The one consumer of this value is `BusinessLayer/ModelContext.cs`:

```csharp
public class ModelContext()
    : ModelContextBase(ConfigurationManager.AppSettings["ConnectionString"]!, ["Entity.Hibernate"]);
```

`ModelContext` exports an `IModelContext` via MEF. It stores the string as the read-only `ConnectionString` property of `ModelContextBase` (in `DataAccessLayer.Hibernate/ModelContextBase.cs`). From there `BusinessLayer/RepositoryFactory.cs` forwards `context.ConnectionString` to the NHibernate factory, which builds the session factory with a SQL Server 2012 provider (`MsSql2012` / Microsoft Data SQL Client driver). So the chain is: `App.config` → `ModelContext` → `ModelContextBase.ConnectionString` → `RepositoryFactory` → NHibernate session factory.

The same connection string value is duplicated in the sibling projects' config files (`ConsoleApp1/App.config`, `WinFormsApp1/App.config`), which serve the other entry points that reference the same `BusinessLayer`.

### Behavior & Details

- The key name is exactly `ConnectionString`; code looks it up by that literal string, so renaming the key breaks the lookup silently.
- `Integrated Security=True` means Windows authentication — no credentials are (or can be) in the string, and the app runs under whatever Windows identity the process has.
- `TrustServerCertificate=True` skips TLS certificate validation, which is a local-development convenience; it would be a problem in any deployment against a production server.
- There is no fallback or default value anywhere in the code; the `!` in `ModelContext` asserts non-null without checking.