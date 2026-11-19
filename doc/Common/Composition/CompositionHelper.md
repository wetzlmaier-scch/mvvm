# `Common/Composition/CompositionHelper.cs`

### Purpose

`CompositionHelper` is the MEF (Module-Extension Framework) bootstrap for the repository's host applications. It scans the application's output directory for loadable assemblies (`.dll` and `.exe`), builds a `CompositionContainer` from all of them, and hands the host a ready-to-use dependency injection container. This single entry point is what lets every host app (`WinFormsApp1`, `Mvvm.Wpf`, `ConsoleApp1`) discover `IModuleController` exports without any manual assembly registration.

### Role in the Codebase

It is the sole file in `Common/Composition/` and the composition root for all three host projects. Each host calls `CreateCompositionContainer` during startup, then resolves `IModuleController` (or other exported types such as `IUserViewModel`) from the returned container and drives the module lifecycle from there. Removing it would break host startup entirely — there is no other path that builds the container.

### Relationships

- **`Common.Reflection.AssemblyHelper`** (`Common/Reflection/AssemblyHelper.cs`) — the only in-repo dependency. `CompositionHelper` uses `AssemblyHelper.Extensions`, which is the constant array `{ ".dll", ".exe" }`, to decide which files in the output directory are candidate assemblies.
- **Framework types** — everything else comes from `System.ComponentModel.Composition` / `Hosting` (the MEF v1 container, catalog, and batch APIs) and `System.Reflection` / `System.IO`.
- **Callers** — `WinFormsApp1/Program.cs` (parameterless overload), `ConsoleApp1/Program.cs` (ignores `ConsoleApp1.exe`), and `Mvvm.Wpf/MvvmApplication.cs` (full three-argument overload, passing module assemblies, ignore list, and search option).

### Behavior & Details

The class is static with three overloads of `CreateCompositionContainer`, all funneling into the full one:

- `CreateCompositionContainer(SearchOption)` — scan defaults, no module assemblies, no ignores.
- `CreateCompositionContainer(string[] ignoreAssemblies, SearchOption)` — same, plus a set of file names to skip.
- `CreateCompositionContainer(string[] moduleAssemblies, string[] ignoreAssemblies, SearchOption)` — the real work.

The full overload proceeds in these steps:

1. **Build the file list.** It locates the directory of `Assembly.GetExecutingAssembly()` — that is, the directory containing the `Common` assembly, which is where all host projects are deployed. It enumerates files with each extension in `AssemblyHelper.Extensions` (`.dll`, `.exe`) using the caller's `searchOption` (default `TopDirectoryOnly`).
2. **Load and catalog each file.** For each file not in the ignore list (matched case-insensitively), it calls `Assembly.LoadFrom` and adds an `AssemblyCatalog` to an `AggregateCatalog`. Duplicate locations are treated as an error, but that `InvalidOperationException` is swallowed by the surrounding `catch (Exception)`, which prints the message to the console and continues. So a broken or conflicting assembly never kills startup; it just produces a console line and disappears.
3. **Guarantee the `Common` assembly is in the catalog.** If the executing assembly's location was not picked up by the directory scan (e.g. it was on the ignore list), it is added explicitly.
4. **Add the entry assembly** (the host's own `.exe`) if present and not yet catalogued.
5. **Load module assemblies** — named assemblies passed by the caller, for example the `Reporting` extension mentioned in the comment. Here behaviour differs from step 2: `Assembly.LoadFrom` is *not* wrapped in the try/catch, and a duplicate location throws `InvalidOperationException` uncaught. Note also that it adds `new AssemblyCatalog(moduleAssembly)` using the *string name* rather than the loaded `Assembly` instance.
6. **Create and pre-compose the container.** It builds a `CompositionContainer` over the aggregate catalog, then composes a `CompositionBatch` containing the container itself (`batch.AddExportedValue(container)`). This makes the container available as an `CompositionContainer` export that other parts can import during composition.

Edge cases worth knowing:

- **Ignore matching is on file name only**, case-insensitively — `["ConsoleApp1.exe"]` skips exactly that file regardless of casing.
- **A directory with no MEF exports still yields a valid container**; the caller is responsible for failing gracefully when `GetExportedValues<IModuleController>()` returns nothing (all three hosts do this).
- **Failures are inconsistent by design of the code**: scan-phase load failures are silent except for a `Console.WriteLine`, while module-assembly-phase failures (duplicate or unloadable named assembly) throw and are *not* caught inside the helper — `MvvmApplication.OnStartup` will surface them as unhandled exceptions.

### Rough edges

- The duplicate-assembly check in the scan loop throws `InvalidOperationException`, but it is always caught by the very next `catch (Exception)` block in the same iteration — so the "already loaded" condition is effectively just another console message, never a real failure, despite being typed as an exception.
- The module-assembly step adds `new AssemblyCatalog(moduleAssembly)` (the string path) instead of `new AssemblyCatalog(assembly)` (the loaded instance) used everywhere else, which re-loads the assembly from its original location.