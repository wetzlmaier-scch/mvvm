# `WinFormsApp1/App.config`

### Purpose

Runtime configuration for the `WinFormsApp1` executable. It supplies a single `appSettings` entry, `ConnectionString`, that points at a local SQL Server instance: `Server=localhost;Database=Users;Integrated Security=True;TrustServerCertificate=True;`. The WinForms app itself never reads this key directly — it exists so the business layer can build its NHibernate session factory when the app starts.

### Role in the Codebase

`App.config` is the standard .NET host configuration file for the `WinFormsApp1` project. At build time the SDK transforms it into `WinFormsApp1.exe.config`, and the .NET configuration system makes its `appSettings` section available to the process at startup via `ConfigurationManager`.

The consumer chain runs through the composition container created in `WinFormsApp1/Program.cs`:

1. `BusinessLayer/ModelContext.cs` reads the value at construction time:
   ```csharp
   : ModelContextBase(ConfigurationManager.AppSettings["ConnectionString"]!, ["Entity.Hibernate"]);
   ```
2. `ModelContext` extends `ModelContextBase` (`DataAccessLayer.Hibernate/ModelContextBase.cs`), which stores the string as the read-only `ConnectionString` property.
3. `RepositoryFactory` (`BusinessLayer/RepositoryFactory.cs`) takes `IModelContext` in its `[ImportingConstructor]` and forwards `context.ConnectionString` into the NHibernate factory, which assumes SQL Server (FluentNHibernate `MsSql2012` configuration).

The WinForms project reaches this chain only indirectly: `WinFormsApp1.csproj` references `UserManagement`, which references `BusinessLayer`. When `Program.Main` builds the `CompositionContainer`, `ModelContext` is among the parts that get constructed, which is the moment the app setting is read.

The same key and value are duplicated in `ConsoleApp1/App.config` and `WpfApp1/App.config` — each host project needs its own copy because the setting is read from the *current* executable's configuration.

### Behavior & Details

- **The key is load-bearing, not optional.** `ModelContext` applies a null-forgiving operator (`!`) to `ConfigurationManager.AppSettings["ConnectionString"]`. If the key is missing from the exe config, the constructor silently receives `null` and the failure surfaces later, when NHibernate tries to open a connection with a null connection string.
- **The value is a literal, not a template.** No transforms or placeholders are used; the string is taken verbatim. Changing the database means editing this file (and the two sibling `App.config` copies if the other hosts should follow).
- **`Integrated Security=True`** means Windows authentication — no credentials in the string — and `TrustServerCertificate=True` suppresses server certificate validation, which matters for a `localhost` dev instance.

### Rough edges

- No fallback exists if the `ConnectionString` key is absent; the `!` in `ModelContext` hides the null until connection time.
- The identical connection string is maintained by hand in three `App.config` files; a change in one does not propagate to the others.