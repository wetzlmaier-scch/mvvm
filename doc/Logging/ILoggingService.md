# `Logging/ILoggingService.cs`

This file defines `ILoggingService`, the single interface the rest of the application logs through. Read it as the logging contract for the whole app: `LoggingService` (in `Logging/LoggingService.cs`) implements it, and `BusinessLayer` and `UserManagement` hold it as an injected property.

### Purpose

`Microsoft.Extensions.Logging.ILogger` — the interface this one extends — logs through `Log<TState>(LogLevel, EventId, TState state, Exception?, Func<TState, Exception?, string> formatter)`. That struct-plus-formatter shape is awkward for everyday calls. `ILoggingService` exists to give callers the plain style everyone writes anyway:

```csharp
LoggingService.LogError(e, "Error saving user");
```

It inherits the framework's three core members — `Log<TState>`, `IsEnabled`, and `BeginScope<TState>` — unchanged, and adds a set of string-message-plus-`params object?[] args` overloads on top.

### Role in the Codebase

This is the public face of the `Logging` project, which contains only two files: this interface and `LoggingService`, its sole implementer. `LoggingService` is `[Export(typeof(ILoggingService))]`, so MEF (`System.ComponentModel.Composition`) hands the Serilog-backed instance to any `[Import]`-marked `ILoggingService` property at composition time.

Because consumers depend on this interface rather than `ILogger` or Serilog directly, the sink engine is swappable: if Serilog were replaced, only `LoggingService` would change. Removing this interface would break `LoggingService`'s declaration and every `ILoggingService` property in `BusinessLayer` and `UserManagement`.

### Relationships

- **Parent type:** `Microsoft.Extensions.Logging.ILogger` (external library, not in this repository). Inherited members: `Log<TState>`, `IsEnabled`, `BeginScope<TState>`.
- **Implementer:** `LoggingService` in `Logging/LoggingService.cs`. Every overload in the interface maps 1:1 to a `LoggingService` method that forwards to an internal `Microsoft.Extensions.Logging.ILogger` built from `SerilogLoggerFactory` (daily-rolling file `Hargassner.Teststand-.log` plus a console sink, minimum level `Debug`). Message formatting and exception rendering therefore behave exactly as in the standard framework.
- **Consumers** (all hold a settable `ILoggingService LoggingService` property, not a constructor parameter):
  - `BusinessLayer/IUserService.cs` / `BusinessLayer/UserService.cs` — the only code that actually logs: `LoggingService.LogError(e, "Error saving user")` in the save-user catch block.
  - `UserManagement/ViewModel/IAddUserViewModel.cs` / `AddUserViewModel.cs` — injected, but no logging call appears in `AddUserViewModel`.
  - `UserManagement/ViewModel/IShowUsersViewModel.cs` / `ShowUsersViewModel.cs` — injected and never referenced.

### Behavior & Details

The interface declares no logic; it is a shape. Its surface is a regular grid:

- **Per-level methods:** `LogTrace`, `LogDebug`, `LogInformation`, `LogWarning`, `LogError`, `LogCritical` — one per `LogLevel`. Each has four overloads that combine an optional `EventId` and an optional `Exception` leading parameter, e.g.:

  ```csharp
  void LogError(EventId eventId, Exception? exception, string? message, params object?[] args);
  void LogError(EventId eventId, string? message, params object?[] args);
  void LogError(Exception? exception, string? message, params object?[] args);
  void LogError(string? message, params object?[] args);
  ```

- **Level-at-runtime methods:** three `Log(LogLevel, ...)` overloads (with `EventId`, with `Exception`, or with neither) for call sites that compute the level. `LoggingService` passes `EventId 0` when none is supplied.

Two things to know before writing or changing code against it:

- A consumer can still call the inherited `Log<TState>` directly, but the intended style is the string/`params` overloads — that is how every current call site logs.
- Nothing here is configurable. The sinks, file name, and minimum level are fixed inside `LoggingService`'s field initializer; the interface offers no hook to change them.

### Rough edges

- `ShowUsersViewModel` and `AddUserViewModel` import `ILoggingService` but never log — dead injections that may mislead a reader into thinking those view models produce log output.
- The grid of 24 per-level overloads is wider than the application uses: only the `LogError(Exception?, string, ...)` shape appears at the one real call site. All `EventId` overloads, and the `EventId`/`Exception` combinations of every other level, are currently unused.