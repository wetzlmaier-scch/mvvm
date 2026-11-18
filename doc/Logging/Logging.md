### Purpose
`Logging.csproj` is the MSBuild project file for the `Logging` project — a small class library that exposes a single logging facade (`ILoggingService`/`LoggingService`) over Serilog, wrapped around `Microsoft.Extensions.Logging`. The project file itself has no logic; it declares the target framework and the packages the logging code needs to compile.

### Role in the Codebase
It is the build definition for the `Logging/` folder, which contains exactly three files: this `.csproj`, `ILoggingService.cs` (the facade interface, extending `Microsoft.Extensions.Logging.ILogger`), and `LoggingService.cs` (the Serilog-backed implementation). Every `.cs` file in `Logging/` is compiled by this project into the `Logging` assembly. Without it, `ILoggingService` and `LoggingService` would not build and the two projects that reference them could not compile.

Notable build settings:
- `TargetFramework` is `net10.0`, and both `ImplicitUsings` and `Nullable` are enabled for all code in the folder.
- Because it uses the plain `Microsoft.NET.Sdk` (not `ProjectReferences`), it exposes no configuration of its own — all logging behavior (log levels, file path, retention) is hard-coded in `LoggingService.cs`, not in a config file.

### Relationships
Two projects reference this one via `<ProjectReference Include="..\Logging\Logging.csproj" />`:
- `BusinessLayer/BusinessLayer.csproj`
- `UserManagement/UserManagement.csproj`

Both inject `ILoggingService` into services and view models (e.g. `BusinessLayer/UserService.cs`, `UserManagement/ViewModel/AddUserViewModel.cs`).

The package references map directly to what the code in this folder imports:
- `Microsoft.Extensions.Logging` and `...Logging.Console` — the `ILogger`/`LogLevel`/`EventId` types that `ILoggingService` extends.
- `Serilog`, `Serilog.Extensions.Logging`, `Serilog.Enrichers.Thread` — the actual logging engine; `LoggingService` builds its `LoggerConfiguration` with thread ID/name enrichers.
- `Serilog.Sinks.File` and `Serilog.Sinks.Debug` — the two sinks (`WriteTo.File` and `WriteTo.Debug`) used by `LoggingService`.
- `System.ComponentModel.Composition` — provides the `[Export(typeof(ILoggingService))]` attribute on `LoggingService`, marking it for composition-based dependency injection.

### Behavior & Details
There is no runtime behavior to document — a `.csproj` is consumed only by MSBuild at build time. The only thing worth knowing is the coupling: the version numbers here (`Serilog 4.3.1`, `Serilog.Extensions.Logging 10.0.0`, `Microsoft.Extensions.Logging 10.0.6`) must stay mutually compatible with the code in `LoggingService.cs`. Bumping a `PackageReference` version in this file is a build-time change that can break compilation or alter log output format for `BusinessLayer` and `UserManagement` alike.