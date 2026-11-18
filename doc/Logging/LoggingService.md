### Purpose

`LoggingService` is the concrete logging implementation of this application. It exists so that business code and view models can log through a single, Serilog-backed facade without touching Serilog or `Microsoft.Extensions.Logging` directly. It is the runtime counterpart to `ILoggingService` (defined in `Logging/ILoggingService.cs`): the interface is what everyone else sees, this class is what actually writes the log lines.

### Role in the Codebase

This class is the single implementer of `ILoggingService` in the repository. It is declared `[Export(typeof(ILoggingService))]` (`System.ComponentModel.Composition`), so it is composed in via MEF rather than constructed manually — callers such as `BusinessLayer/UserService.cs` and `UserManagement/ViewModel/AddUserViewModel.cs` just declare an `ILoggingService` property and receive this instance at runtime. Removing it would break the MEF composition for every class that imports `ILoggingService`.

### Relationships

- **Implements `ILoggingService`** (in `Logging/ILoggingService.cs`), which itself extends `Microsoft.Extensions.Logging.ILogger`. So this class must satisfy two sets of members: the `ILogger` contract (`Log<TState>`, `IsEnabled`, `BeginScope`) and the interface's per-level convenience methods.
- **Wraps `Microsoft.Extensions.Logging.ILogger`** privately, produced through `Serilog.Extensions.Logging`'s `SerilogLoggerFactory` over a configured Serilog `Logger`. All methods are one-line delegations to that field.
- **Callers** log through the interface only; the concrete class is referenced nowhere outside its own file.

### Behavior & Details

The entire configuration lives in the initializer of the `_logger` field:

- **Enrichers:** each log event is enriched with the thread ID, thread name, and a `CategoryName` property. The category name is set via the private nested `CategoryNameEnricher` to `string.Empty` — so every event carries `CategoryName` = "" (see "Rough edges" below).
- **Minimum level:** `Debug` — `Trace` messages are filtered out before they reach any sink, even though `LogTrace` methods exist.
- **Sinks:**
  - File: `Hargassner.Teststand-.log` (a literal filename; no date component in the name — the date is supplied by Serilog's daily rolling). Daily rolling, keeping the last 5 files.
  - Debug window, with an explicit template: `{Timestamp:HH:mm:ss} [{Level}] [{CategoryName}] [{ThreadId}] [{ThreadName}] {Message}{NewLine}{Exception}`.
- **The logger is created with category `string.Empty`** (`CreateLogger(string.Empty)`), so no per-caller category distinction is possible — all output shares one category.

The members then split into two groups:

- **`ILogger` members** — `Log<TState>`, `IsEnabled`, and `BeginScope` are pure pass-throughs to the wrapped logger.
- **Convenience methods** — one set of four `Log{Debug,Trace,Information,Warning,Error,Critical}` overloads (with/without `EventId`, with/without `Exception`), plus three `Log(LogLevel, ...)` overloads. Each is a one-line call to `_logger.Log(...)` at the fixed level. The `Log(LogLevel, ...)` overloads with no `EventId` pass `EventId` 0.

Since every method is a delegation, there is no state, no retry, no buffering of its own; behaviour is entirely Serilog's.

### Rough edges

- `CategoryNameEnricher` is constructed with `string.Empty`, so the `CategoryName` property it adds is always empty — the enricher (and the `{CategoryName}` placeholder in the Debug template) currently adds noise rather than information.
- The file sink path `Hargassner.Teststand-.log` is a hard-coded literal, so all output goes to one file regardless of which class logged it, and the `CreateLogger(string.Empty)` category is similarly a constant, making per-source filtering impossible.
- `LogTrace` methods exist, but the configured `MinimumLevel.Debug()` discards trace events, so they silently write nothing.
- The logger is built once per `LoggingService` instance in a field initializer; there is no way to reconfigure sinks or levels after construction.