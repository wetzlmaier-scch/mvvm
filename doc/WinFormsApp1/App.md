# App.config

**Path:** `WinFormsApp1/App.config`

### Purpose

This is the .NET application configuration file for the WinForms client. Its one job is to supply the SQL Server connection string that the data-access layer uses to reach the `Users` database.

### Role in the Codebase

`App.config` is the runtime configuration file for the `WinFormsApp1` project. At build time it is transformed into `WinFormsApp1.exe.config` and read at startup by the .NET configuration system. It holds a single `<appSettings>` entry, `ConnectionString`, whose value is:

```
Server=localhost;Database=Users;Integrated Security=True;TrustServerCertificate=True;
```

This points at a local SQL Server instance (`localhost`), targets the `Users` database, authenticates via Windows credentials (`Integrated Security=True`), and disables server-certificate validation (`TrustServerCertificate=True`) — typical for a local development setup.

If the file or the key were removed, the application would fail to start: the consumer reads the setting with a non-null-asserted call (see below), so a missing key throws rather than degrading gracefully.

### Relationships

The value is consumed by `BusinessLayer/ModelContext.cs`, which does:

```csharp
public class ModelContext()
    : ModelContextBase(ConfigurationManager.AppSettings["ConnectionString"]!, ["Entity.Hibernate"]);
```

`ModelContext` passes the string to `DataAccessLayer.Hibernate/ModelContextBase.cs`, which stores it as the `ConnectionString` property. From there it flows to `RepositoryFactory` and `SessionFactoryBuilder` in `DataAccessLayer.Hibernate`, where it is used to build the NHibernate session factory. The WinForms `Program.cs` pulls `ModelContext` in via MEF (`[Export(typeof(IModelContext))]`), so this config file is the root of the whole data-access chain for this client.

Note the same connection-string value is duplicated verbatim in `ConsoleApp1/App.config` and `WpfApp1/App.config` — the three front-end projects in this solution each carry their own copy, so changing the database target requires editing each one.

### Behavior & Details

- The file contains only one `appSettings` key; there are no connectionStrings sections, binding redirects, or custom sections.
- Because `ModelContext` uses `ConfigurationManager.AppSettings["ConnectionString"]!` with a null-forgiving operator, the application assumes the key always exists. There is no fallback value anywhere — removing or mistyping the key produces a `NullReferenceException` at startup when the `ModelContext` singleton is first created.

### Rough edges

- The connection string is hardcoded to `localhost` in three separate `App.config` files (WinForms, WPF, Console). Any environment change (different server, credentials, database name) must be made in every copy, and they have already drifted as a pattern risk rather than currently.