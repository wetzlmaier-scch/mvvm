### Purpose

`Common/ExitCodes.cs` defines the process exit codes the application returns when it cannot run. It exists so that the several application entry points in this solution report their outcomes with named, consistent values instead of bare integers, letting anything inspecting the process (a test harness, a supervisor, a developer reading a return code) tell "started fine" apart from "failed to start" without magic numbers.

### Role in the Codebase

The class lives in the `Common` project, the shared utility project every other project in the solution references. It is a small helper, not an architectural component: a `public static class` with three compile-time constants. Nothing else in `Common` depends on it; it only depends in the reverse direction — the host applications consume its constants when they must terminate.

If it were removed, the two hosts that reference it (`WinFormsApp1/Program.cs`, `Mvvm.Wpf/MvvmApplication.cs`) would fail to compile, since they reference the constants directly rather than hard-coding the numbers.

### Relationships

- **`Common/ExitCodes.cs`** (this file) — defines the constants in namespace `Common`. No imports, no base types.
- **`WinFormsApp1/Program.cs`** — the WinForms host's `Main` returns `ExitCodes.Ok` on successful module startup and `ExitCodes.Error` when module loading fails (`ReflectionTypeLoadException`) or no `IModuleController` is found.
- **`Mvvm.Wpf/MvvmApplication.cs`** — the WPF host's `OnStartup` calls `Current.Shutdown(ExitCodes.Error)` in the same two failure cases.

`ConsoleApp1/Program.cs` is a host too but never references these constants; it signals failure by throwing instead of returning a code.

### Behavior & Details

Three constants, all compile-time (`const`), so their use in return statements and `Shutdown` calls is resolved at compile time:

- `Ok = 0` — the conventional "success" exit status, returned by `WinFormsApp1`'s `Main` after all modules have been initialized and started.
- `Error = -1` — a generic "startup failed" status, used when the MEF composition container cannot load module types or no module controllers were exported.
- `InsufficientPrivileges = -2` — currently defined but not referenced anywhere else in the repository. It is available for a host to return when it needs to distinguish a permission failure from a generic error, but no host does so today.

Two caveats worth knowing:

- The negative values are not what an OS process actually reports. On Windows, process exit codes are 32-bit unsigned values, so a process exiting with `-1` surfaces as `0xFFFFFFFF` (4294967295) and `-2` as `4294967294`. Any external code that checks the exit code must either compare against those wrapped values or treat "anything non-zero" as failure. Within the process, the constants are fine.
- Only `Ok` and `Error` are in use. The constant set is a fixed vocabulary — a new failure class (e.g. using `InsufficientPrivileges`) requires every host's failure paths to agree on which value to return, since the callers are not centralized.

### Rough edges

`InsufficientPrivileges` has no callers in the repository, so it is either reserved for a host that does not (yet) exist in this solution or dead code. There is no central place where a host maps a specific failure to one of these codes — each entry point picks the constant inline, so the vocabulary is only as consistent as the hosts' discipline.