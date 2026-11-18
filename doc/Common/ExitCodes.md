### Purpose

`Common/ExitCodes.cs` (in the `Common` namespace) defines the process exit codes the application hosts return when they terminate. It exists so the two host applications in the solution report success and failure to the operating system with a shared, named vocabulary instead of scattered magic numbers — `Main` returning `0` on success and `-1` when module loading fails.

The class `ExitCodes` is a `public static class` of three `const int` members. It has no base type, no state, and no dependencies.

### Role in the Codebase

It is one of several small helper types under `Common/` (alongside `Composition/`, `Converter/`, `Diagnostics/`, `Reflection/`, and `Windows/`) compiled into the shared `Common` assembly (`Common.csproj`). The hosts reference that project and consume these constants from their entry points. Removing the file would break compilation of the entry points below; nothing else in the repository uses it.

### Relationships

- `WinFormsApp1/Program.cs` — the WinForms host's `Main` returns `ExitCodes.Ok` after the module message loop ends, and `ExitCodes.Error` in two failure paths: a caught `ReflectionTypeLoadException` from `GetExportedValues<IModuleController>()`, and the zero-modules case (which also shows a "No module found." dialog).
- `Mvvm.Wpf/MvvmApplication.cs` — the WPF host's `OnStartup` calls `Current.Shutdown(ExitCodes.Error)` in the same two failure cases; it never shuts down with `ExitCodes.Ok` on success.

### Behavior & Details

- `Ok = 0` — the standard success code; the only constant the hosts use for a normal exit.
- `Error = -1` — the generic failure code, used by both hosts for every fatal startup path. Note that `-1` is not a conventional OS exit code (process exit codes are usually 32-bit values where `0` means success); the hosts return or pass it as-is, so external consumers of these exit codes inherit whatever the platform truncates it to.
- `InsufficientPrivileges = -2` — defined but never referenced by any code in this repository. It is available for a host that needs to distinguish a permission failure from a generic one, but no current caller does.

Because the constants are `const int`, values are inlined at compile time into the consuming hosts — there is no runtime lookup, and changing a value requires recompiling the hosts.

### Rough edges

`InsufficientPrivileges` has no callers in the repository, so it is either reserved for a host that does not (yet) exist in this solution or dead code. The negative values (`-1`, `-2`) are unusual for process exit codes and depend on the hosts passing them through unchanged.