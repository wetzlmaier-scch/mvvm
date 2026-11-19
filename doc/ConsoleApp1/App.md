# ConsoleApp1/App.config

### Purpose

This is the runtime configuration file for the `ConsoleApp1` console application. It holds a single `appSettings` entry, `ConnectionString`, that tells the data-access layer where the `Users` database lives: a local SQL Server instance (`Server=localhost`) using Windows integrated security. Without it, the application cannot build its NHibernate session factory and all user-management operations fail at runtime.

### Role in the Codebase

`ConsoleApp1` is a small console program (`Program.cs`) that drives the shared `BusinessLayer` and `UserManagement` projects: it adds 100 test users and then prints them all out. The program itself never reads configuration directly — the setting is a passive dependency picked up by the shared business layer when the MEF composition container is built. The same pattern and the same connection string appear in the sibling hosts `WinFormsApp1/App.config` and `WpfApp1/App.config`; each executable carries its own copy, so changing the database for this app means editing only this file.

At build time, the .NET SDK copies it to `ConsoleApp1.exe.config` next to the executable, where the standard .NET configuration system (`System.Configuration.ConfigurationManager`) locates it at runtime.

### Relationships

- **Consumer:** `BusinessLayer/ModelContext.cs` — the class marked `[Export(typeof(IModelContext))]` whose primary constructor argument is `ConfigurationManager.AppSettings["ConnectionString"]!`. That value flows into `ModelContextBase` (in `DataAccessLayer.Hibernate/ModelContext.cs`) as the read-only `ConnectionString` property, is imported by `BusinessLayer/RepositoryFactory.cs` through its `[ImportingConstructor]`, and ultimately reaches the NHibernate `SessionFactoryBuilder`.
- **Key name is load-bearing:** the lookup uses the literal string `"ConnectionString"`, so renaming the key breaks the lookup. The value itself must remain a SQL Server connection string — the driver and dialect are fixed in the data-access layer, not configured here.

### Behavior & Details

- The `appSettings` section is read once, when the MEF container is constructed in `Program.Main` (via `CompositionHelper.CreateCompositionContainer`). The console app then reuses that single connection for every `SaveUserAsync` and `OnLoadedAsync` call.
- There is no fallback if the key is missing: `ModelContext` applies a null-forgiving operator to the lookup, which silences the compiler. An absent `ConnectionString` surfaces later as a null-connection failure deep in NHibernate, not as a clear configuration error at startup.
- The value uses `Integrated Security=True` (Windows authentication, no username/password in config) and `TrustServerCertificate=True`, which skips TLS certificate validation — typical for a local development server, but a value to review before pointing the connection at anything untrusted.