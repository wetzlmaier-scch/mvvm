# Mvvm.Console/Mvvm.Console.csproj

### Purpose

This is the MSBuild project file for the `Mvvm.Console` library. It defines a class library targeting `net10.0-windows` that provides a console-based implementation of the `IMessageService` abstraction from the `Mvvm` project — i.e., the way view models display messages, warnings, errors, and yes/no questions to a console user.

It exists so the solution can have a platform-specific messaging backend for console hosts, parallel to the WPF and WinForms equivalents.

### Role in the Codebase

This project file is the build manifest for a one-class library (`Mvvm.Console.MessageService`). It is referenced by:

- `ConsoleApp1/ConsoleApp1.csproj` — the console application that composes and runs the view models.
- `Demo.slnx` — the solution file, which includes it as a project.

If the project were removed, `ConsoleApp1` could not build (it has a direct `ProjectReference`), and the console host would have no `IMessageService` export for MEF to discover.

### Relationships

- **`Mvvm/Mvvm.csproj`** — project reference. The `Mvvm` project defines `IMessageService` (in `Mvvm/IMessageService.cs`) and `MessageResult`; this library implements that interface.
- **`System.ComponentModel.Composition` (NuGet, 10.0.6)** — the MEF package that supplies the `[Export]` attribute used by `MessageService`. The same package version is also referenced by the `Mvvm` project itself, which declares the `[Import]` side.
- **Consumers:** `ConsoleApp1` pulls in this project so that MEF can discover `MessageService` and inject it wherever `IMessageService` is imported.

### Behavior & Details

- **Target framework `net10.0-windows`:** the same TFM as the other projects in the solution. No `OutputType` element is present, so MSBuild defaults to a class library (`Library`).
- **`ImplicitUsings` / `Nullable`:** enabled for this project, matching the rest of the solution's projects.
- **Two `ItemGroup` blocks:** one for the MEF NuGet package, one for the project reference. Nothing else (no output type, no content, no analyzers).

There is no build-time logic, packaging, or conditional behavior in this file beyond the above.