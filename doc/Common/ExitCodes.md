# Common/ExitCodes.cs

### Purpose

`Common/ExitCodes.cs` (in the `Common` namespace) defines the process exit codes the application hosts return when they terminate. It exists so the host applications in the solution report success and failure with a single, shared convention instead of scattering ad-hoc integers (or the conventional non-zero `1`) through their entry points.

The class `ExitCodes` is a `public static class` holding three `const int` members. It has no base type, no state, and no dependencies.

### Role in the Codebase

This file is a shared leaf constant inside the `Common` project, which holds cross-cutting helpers (`Composition/`, `Converter/`, `Diagnostics/`, `Reflection/`, `Windows/`) consumed by the host applications and the Mvvm host libraries. `ExitCodes` is referenced by the two executable hosts in the solution — the WinForms and WPF applications — and is invisible to the business/data layers, which do not return process exit codes.

### Relationships

- `WinFormsApp1/Program.cs` — the WinForms host's `Main` returns `ExitCodes.Ok` after the module message loop ends, and `ExitCodes.Error` in two failure paths: a caught `ReflectionTypeLoadException` during module resolution, and zero exported `IModuleController` values.
- `Mvvm.Wpf/MvvmApplication.cs` — the WPF host's `OnStartup` calls `Current.Shutdown(ExitCodes.Error)` in the same two failure cases; it never shuts down with `ExitCodes.Ok` on success.

### Behavior & Details

| Constant | Value | Meaning |
|---|---|---|
| `Ok` | `0` | Normal completion; returned by `Main` when all modules ran. |
| `Error` | `-1` | Fatal startup failure (module assembly failed to load, or no modules found). |
| `InsufficientPrivileges` | `-2` | Defined for the codebase's privilege-failure convention, but not returned by any host in this repository. |

All three are compile-time constants in a `static` class, so callers use them unqualified via `using Common;` and the compiler inlines the values.

One caveat for anyone changing values: the WPF host passes `ExitCodes.Error` to `Application.Shutdown(int)`, and the WinForms host returns these values from `int Main()`. On Windows, process exit codes are 32-bit and are commonly displayed as unsigned, so the negative values surface as large positive numbers (e.g. `-1` becomes `4294967295` to `Task.Wait()`/`Process.ExitCode` consumers). Any tooling that checks exit codes must compare against the signed constants, not the raw unsigned number.

Rough edges: `InsufficientPrivileges` is defined but never referenced by any host, making it currently dead code.