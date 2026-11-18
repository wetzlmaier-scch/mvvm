# `Logging/ILoggingService.cs`

### Purpose

This file defines `ILoggingService`, the logging contract that the rest of the application logs through. It extends `Microsoft.Extensions.Logging.ILogger` (an external library type, not defined in this repository) with one argument-style improvement: instead of the framework's struct-based `Log<TState>(LogLevel, EventId, TState, ...)` call shape, it offers string-message-plus-`params object?[] args` overloads. A caller can write `LoggingService.LogError(e, "Error saving user")` instead of dealing with `TState` and a formatter delegate.

For each of the six `LogLevel` values — `LogDebug`, `LogTrace`, `LogInformation`, `LogWarning`, `LogError`, `LogCritical` — the interface declares four overloads, each of which is an optional combination of `EventId` and `Exception` leading parameters, e.g.:

```csharp
void LogError(EventId eventId, Exception? exception, string? message, params object?[] args);
void LogError(EventId eventId, string? message, params object?[] args);
void LogError(Exception? exception, string? message, params object?[] args);
void LogError(string? message, params object?[] args);
```

It also declares three `Log` overloads that take the level explicitly (`Log(LogLevel, ...)`, `Log(LogLevel, EventId, ...)`, `Log(LogLevel, Exception?, ...)`) for cases where the level is not known at the call site.

### Role in the Codebase

This is the facade of the `Logging` project, a small class library. The concrete implementation is `LoggingService` in `Logging/LoggingService.cs`, which is backed by Serilog (rolling daily file plus a console sink). `LoggingService` carries `[Export(typeof(ILoggingService))]` and is wired to consumers via MEF (`System.ComponentModel.Composition`), so any class with an `[Import]`-marked `ILoggingService` property receives the Serilog-backed instance at composition time.

Consumers depend on this interface rather than `ILogger` directly, which keeps the MEF export point stable: if the underlying engine or sinks change, only `LoggingService` changes. If this interface were removed, `LoggingService` would have nothing to implement and every injected property in `BusinessLayer` and `UserManagement` would break.

### Relationships

- **Parent type:** `Microsoft.Extensions.Logging.ILogger` (external library). The interface inherits its three core members — `Log<TState>`, `IsEnabled`, `BeginScope<TState>` — which `LoggingService` forwards to a `Microsoft.Extensions.Logging.ILogger` created from `SerilogLoggerFactory`.
- **Implementation:** `Logging/LoggingService.cs` — the only implementer. Each convenience overload there forwards to the internal `_logger` via `Microsoft.Extensions.Logging.LoggerExtensions.Log` overloads, so message formatting (`{message}` with `args`) and exception handling behave exactly as in the standard framework.
- **Consumers** (all via MEF-injected `ILoggingService` properties):
  - `BusinessLayer/IUserService.cs` and `BusinessLayer/UserService.cs` — `UserService` calls `LoggingService.LogError(e, "Error saving user")` when a user save fails.
  - `UserManagement/ViewModel/IAddUserViewModel.cs` and `AddUserViewModel.cs` — injected and available to the save-command handler.
  - `UserManagement/ViewModel/IShowUsersViewModel.cs` and `ShowUsersViewModel.cs` — injected, but no call to `LoggingService` appears in `ShowUsersViewModel`, so the import is currently unused there.

### Behavior & Details

- The interface adds no logic of its own; all behaviour lives in `LoggingService`. Notably, `LoggingService` creates its Serilog `ILogger` in a field initializer (not through MEF or a factory), so the sink configuration — output file `Hargassner.Teststand-.log`, `RollingInterval.Day`, retaining 5 files, minimum level `Debug` — is fixed at class construction and not configurable through this interface.
- Because the interface extends `ILogger`, a consumer can still call the framework's `Log<TState>` method directly, but the intended call style is the string/`params` overloads, which match how every existing call site logs.
- The `Log(LogLevel, ...)` overloads let a single call site pick the level at runtime; `LoggingService` passes `EventId 0` (or the given `EventId`) when none is supplied.

### Rough edges

- `ShowUsersViewModel` imports `ILoggingService` but never uses it — dead injection that a reader may mistake for logging that actually happens there.
- The interface declares 27 overloads; since `EventId` is `params`-free and the `Exception`/`EventId` overloads never appear at any current call site, the surface is wider than the application actually uses.