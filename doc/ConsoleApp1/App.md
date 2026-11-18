### Purpose

`App.config` is the runtime configuration for the `ConsoleApp1` project. It supplies the single `ConnectionString` app setting — a SQL Server connection to a `Users` database on `localhost` using Windows integrated security. The console app itself never reads this setting; it exists because the composition-root code pulled into the app does.

### Role in the Codebase

The file is consumed only through .NET's configuration system. At build time it becomes `ConsoleApp1.exe.config`, and at runtime `System.Configuration.ConfigurationManager.AppSettings["ConnectionString"]` resolves against it. That read happens in `ModelContext` (`BusinessLayer/ModelContext.cs`), which is exported as `IModelContext` and composed into `ConsoleApp1`'s MEF container.

Without this file (or the key in it), `ModelContext`'s constructor receives `null` for the connection string, and the failure surfaces later when the NHibernate session factory tries to open a connection. There is no fallback value anywhere in the codebase.

The same connection string is duplicated verbatim in `WinFormsApp1/App.config` and `WpfApp1/App.config` — each host project carries its own copy rather than sharing one.

### Relationships

- **`BusinessLayer/ModelContext.cs`** — the direct consumer. Its primary constructor argument is `ConfigurationManager.AppSettings["ConnectionString"]!`, passed (with a null-forgiving operator) to `ModelContextBase` in `DataAccessLayer.Hibernate/ModelContextBase.cs`.
- **`DataAccessLayer.Hibernate/HibernateRepositoryFactory.cs`** — inherits the stored connection string via `IModelContext` and passes it to FluentNHibernate's `MsSql2012` configuration, so the value ultimately determines which database `SaveUserAsync()` and the user queries in `Program.cs` talk to.
- **`DataAccessLayer.Hibernate/SessionFactoryBuilder.cs`** — also takes a connection string, but is currently dead code; it does not affect this app's behaviour.

### Behavior & Details

- The value is `Server=localhost;Database=Users;Integrated Security=True;TrustServerCertificate=True;`. The `Integrated Security=True` part means the app connects as the Windows account running the process — no username or password lives in the file.
- `TrustServerCertificate=True` suppresses SQL Server certificate validation. This is a local-development convenience, not something you want in a deployed environment.
- Only this one key exists; there are no other `appSettings`, `connectionStrings` sections, or config switches.
- Changing `Server`, `Database`, or the security mode here changes the database every layer in the composition graph talks to. There is no environment-specific variant (no `App.Development.config` etc.) in this project.

### Rough edges

- The connection string is copy-pasted across three `App.config` files, so pointing the apps at a different database requires editing all three.
- `ModelContext` assumes the key exists and uses `!` on the lookup; a missing setting fails only at connection time, far from its cause.