# Common/Common.csproj

`Common/Common.csproj` — the MSBuild project file for the `Common` class library in `Demo.slnx`.

### Purpose

Build definition for the shared `Common` project. It targets `net10.0-windows` and is configured as a WPF project (`UseWPF`), so the project can reference WPF/XAML facilities even though it is a class library, not an application. It also enables nullable reference types and implicit usings.

The single package dependency is `System.ComponentModel.Composition` (v10.0.6), which provides the MEF (Managed Extensibility Framework) types used by `Common/Composition/CompositionHelper.cs` to discover and compose plugin assemblies at runtime. Note this package is referenced at the same version by several sibling projects (`Mvvm`, `Mvvm.Console`, `Mvvm.Wpf`, `Logging`, `UserManagement`) — the MEF-based plugin architecture spans the solution, and this file is the piece of it that lets the `Common` MEF helpers compile.

### Role in the Codebase

This is a build manifest, not application code. Removing it would drop the `Common` project from `Demo.slnx` and break the four projects that reference it (`ConsoleApp1`, `Mvvm`, `WinFormsApp1`, `WpfApp1`). It has no runtime behavior of its own.

### Relationships

- **Produces**: the `Common` assembly, containing the helpers under `Common/` (`Composition/CompositionHelper.cs`, `Reflection/AssemblyHelper.cs`, `Diagnostics/ExtendedGuard.cs`, `Converter/DebugConverter.cs`, `ExitCodes.cs`).
- **Consumed by**: `ConsoleApp1`, `Mvvm`, `WinFormsApp1`, and `WpfApp1` via project references.
- **External dependency**: the `System.ComponentModel.Composition` NuGet package — the only one.

### Behavior & Details

- `TargetFramework` is `net10.0-windows`, so the assembly can only load in Windows-targeted hosts; every dependent project inherits this constraint transitively.
- `UseWPF=true` pulls in WPF framework references for a library project, which is what allows this assembly to be shared with the WPF apps (`WpfApp1`, `Mvvm.Wpf` ecosystem) in the solution.
- Because the `System.ComponentModel.Composition` reference lives here as well as in the Mvvm/Logging/UserManagement projects, changing the MEF package version should be done consistently across all of those csproj files to avoid type-identity mismatches in the composition container.

### Rough edges

- The package pinning of `System.ComponentModel.Composition` is duplicated across at least six project files rather than centralized (e.g., via Directory.Packages.props), so version drift is easy.
- `UseWPF=true` on a shared class library makes WPF a hard build-time dependency for all four consuming projects, console or WinForms included.