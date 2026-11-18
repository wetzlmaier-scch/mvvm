I have everything I need. Here is the documentation.

---

# `Logging/ILoggingService.cs`

### Purpose

This interface is the project's logging contract. It extends `Microsoft.Extensions.Logging.ILogger` with a set of convenience methods so callers can write a log line in one call — `logger.LogWarning("Failed to save {Id}", id)` — without building an `EventId`, choosing a level-specific `ILogger.Log` overload, or wrapping anything. The reason it exists alongside `ILogger` is ergonomics: `ILogger` only exposes the generic `Log<TState>(LogLevel, EventId, TState, Exception?, formatter)` method, which is awkward to call by hand. This interface turns "log this at this level, maybe with this exception" into a directly callable method for each level.

### Role in the Codebase

`ILoggingService` is the abstraction that the rest of the application logs through; `Logging/LoggingService.cs` is its concrete implementation (backed by Serilog, writing to a rolling daily file and to the debug console). The interface is injected via MEF — `LoggingService` is marked `[Export(typeof(ILoggingService))]` — and the consuming classes declare an `ILoggingService LoggingService { get; set; }` property and import it.

Callers found in the repository that depend on it:

- `BusinessLayer/UserService.cs` (via `BusinessLayer/IUserService.cs`) — e.g. `LoggingService.LogError(e, "Error saving user")` in its save path
- `UserManagement/ViewModel/AddUserViewModel.cs` (via `IAddUserViewModel`)
- `UserManagement/ViewModel/ShowUsersViewModel.cs` (via `IShowUsersViewModel`)

If the interface were removed, those classes could no longer declare their logger dependency, and the MEF export/import contract between `LoggingService` and its consumers would break.

### Relationships

- **Base type:** `Microsoft.Extensions.Logging.ILogger` (from the Microsoft.Extensions.Logging package, not this repo). That is why the interface must also be implemented for `Log<TState>`, `IsEnabled`, and `BeginScope<TState>` — those come from `ILogger` and are satisfied by `LoggingService` delegating to its internal `ILogger`, not by the convenience methods declared here.
- **Implementation:** `Logging/LoggingService.cs` (`LoggingService : ILoggingService`).
- **Dependency injection:** MEF (`System.ComponentModel.Composition`). The `[Export(typeof(ILoggingService))]` on `LoggingService` and the `ILoggingService` properties on the consuming classes are how it is wired up.
- **Callers** assign messages and exceptions directly, e.g. `LoggingService.LogError(e, "Error saving user")`, which maps to the `LogError(Exception?, string?, params object?[])` overload.

### Behavior & Details

The interface declares no state or logic — it is purely a set of method signatures — but the shape of those signatures is the contract the whole codebase programs against.

For each of the six `LogLevel` values — `LogTrace`, `LogDebug`, `LogInformation`, `LogWarning`, `LogError`, `LogCritical` — there are four overloads, varying in which optional pieces a caller supplies:

1. `(EventId eventId, Exception? exception, string? message, params object?[] args)` — full form
2. `(EventId eventId, string? message, params object?[] args)` — with an event id but no exception
3. `(Exception? exception, string? message, params object?[] args)` — with an exception but no event id
4. `(string? message, params object?[] args)` — message only

`message` uses standard string-formatting placeholders (`{0}`, `{1}`, …) filled by `args`, the same convention as the underlying `ILogger`. All parameters except `message` are nullable; a caller may pass `null` for the exception, the event id, or even the message.

In addition, three `Log` overloads take the `LogLevel` explicitly for cases where the level is not fixed at compile time: `Log(LogLevel, string?, params object?[])`, `Log(LogLevel, EventId, string?, params object?[])`, and `Log(LogLevel, Exception?, string?, params object?[])`.

Because it extends `ILogger`, a consumer of this interface can also use the generic `Log<TState>(LogLevel, EventId, TState, Exception?, Func<TState, Exception?, string>)`, `IsEnabled(LogLevel)`, and `BeginScope<TState>(TState)` — the implementation delegates those to its inner Serilog-backed `ILogger`.

One thing to note for maintainers: the overload sets are not uniform across the `Log` family — the explicit-level `Log` overloads lack the `(LogLevel, EventId, Exception?, string?, …)` combination, so a caller wanting both an event id and an exception at a dynamic level must drop down to the inherited `ILogger.Log<TState>` form.

### Rough edges

- The four-overload pattern is repeated six times, one per level, and the explicit-level `Log` overloads omit a few combinations (e.g. no `EventId` + `Exception` variant). Adding a new combination requires touching both the interface and `LoggingService`.
- `message` is typed `string?` yet the methods are used as format strings; there is no `IFormatProvider`/culture parameter, so formatting follows the default culture.