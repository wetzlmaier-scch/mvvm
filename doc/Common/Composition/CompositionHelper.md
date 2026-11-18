# `Common/Composition/CompositionHelper.cs`

### Purpose

`CompositionHelper` is a static class that builds a MEF (`System.ComponentModel.Composition`) `CompositionContainer` from every loadable assembly in the application's directory. It exists so the host applications in this repository (`WinFormsApp1`, `WpfApp1` via `Mvvm.Wpf`, `ConsoleApp1`) can bootstrap their module architecture without each one reimplementing the same assembly-discovery-and-cataloging logic. The returned container is what the hosts query for their `IModuleController` exports, i.e. the application's plugins.

### Role in the Codebase

`CompositionHelper` sits in the `Common` project's `Composition` subdirectory and is the sole file there. It is the shared bootstrapping helper: every host project references `Common` and calls `CreateCompositionContainer(...)` at startup, before resolving any exports. If it were removed, each host would have to build its own `AggregateCatalog` from disk, and none of the hosts' module discovery would work as-is. It is pure plumbing — it contains no domain logic of its own; all behaviour flows through the container it returns.

### Relationships

- **`Common.Reflection.AssemblyHelper`** (`Common/Reflection/AssemblyHelper.cs`) — the only in-repo dependency. `CompositionHelper` uses `AssemblyHelper.Extensions` (the constant `{ ".dll", ".exe" }`) to decide which file types count as candidate assemblies.
- **`System.ComponentModel.Composition.Hosting`** (NuGet package `System.ComponentModel.Composition`, pinned to 10.0.6 in the host projects) — source of `CompositionContainer`, `AggregateCatalog`, `AssemblyCatalog`, `CompositionBatch`. This is the framework being wrapped, not code in this repository.
- **Callers** —
  - `WinFormsApp1/Program.cs` calls the parameterless `CreateCompositionContainer()` in `Main()`, then resolves `IModuleController` exports and drives `Initialize()`/`Run()` on them.
  - `Mvvm.Wpf/MvvmApplication.cs` calls the full overload in `OnStartup`, passing module and ignore assembly lists through from its constructor, then does the same `Initialize()`/`Run()` sequence and additionally `Shutdown()`s each controller in `OnExit` and disposes the container.
  - `ConsoleApp1/Program.cs` calls `CreateCompositionContainer(["ConsoleApp1.exe"])` — i.e. the *ignore* overload — which is a mistake; see Rough edges.
- **What the hosts depend on from the container:** `IModuleController` (defined in `Mvvm/IModuleController.cs`), exported by classes like `WinFormsApp1/Controller/DefaultModuleController.cs` and `WpfApp1/Controller/DefaultModuleController.cs`. Those classes are invisible to this file; it only guarantees they become discoverable by being placed in a catalog that the container can see.

### Behavior & Details

There are three overloads of `CreateCompositionContainer`, all static, all returning a ready-to-use `CompositionContainer`:

- `CreateCompositionContainer()` — no ignore list, top-level directory only.
- `CreateCompositionContainer(string[] ignoreAssemblies, SearchOption)` — same, but skips the named assemblies.
- `CreateCompositionContainer(string[] moduleAssemblies, string[] ignoreAssemblies, SearchOption)` — the real implementation; the other two delegate to it with empty arrays.

The full implementation proceeds in five steps:

1. **Enumerate candidate files.** It takes the directory of `Assembly.GetExecutingAssembly().Location` (the `Common` assembly's own directory, which in the hosts is the same as the output directory) and enumerates every `*.dll` and `*.exe` matching the given `SearchOption` (default `TopDirectoryOnly`). File names are compared against `ignoreAssemblies` case-insensitively; ignored files are skipped entirely.

2. **Load each assembly and add it to an `AggregateCatalog`.** Each candidate is `Assembly.LoadFrom`'d and wrapped in an `AssemblyCatalog`. A `HashSet<string>` of lower-cased `assembly.Location` values prevents the same physical assembly from being cataloged twice; a duplicate triggers an `InvalidOperationException`, which is swallowed by the `catch (Exception)` that surrounds the whole load. On any failure (bad PE, load conflict, etc.) only `ex.Message` is written to `Console` and the loop continues — so a single broken DLL does not stop startup, but it also does not surface any stack trace.

3. **Ensure the executing assembly is cataloged.** If `Common`'s own assembly was not picked up in step 2 (which can happen if its file was in the ignore list), it is added explicitly, so exports that live in `Common` itself remain reachable.

4. **Add the entry assembly** (`Assembly.GetEntryAssembly()`) if present and not yet cataloged, so the host project's own exports (e.g. `DefaultModuleController`) are visible to the container.

5. **Load the explicit `moduleAssemblies`.** These are passed as file paths to `Assembly.LoadFrom`. A duplicate check throws `InvalidOperationException("Assembly is already loaded.")` — but note the catalog entry is added with `new AssemblyCatalog(moduleAssembly)`, passing the *path string* again rather than the already-loaded `Assembly` instance, unlike step 2. (See Rough edges.)

Finally it constructs the `CompositionContainer` over the aggregate catalog, and self-composes one value: the container itself, exported via a `CompositionBatch`. This is the mechanism that lets any MEF part in any loaded assembly write `Import CompositionContainer` to get a reference to the container (e.g. for runtime composition). Without this batch no exports would be discoverable, because at least one composition pass has to be triggered.

Notable edge cases and caveats:

- **Directory scope is fixed.** Discovery is anchored to `Common`'s location, not the process's working directory. This means the helper assumes `Common.dll` sits next to the other assemblies — true for all current hosts, but it would silently find nothing useful if the layout ever changed.
- **Error handling is console-only.** A WPF host calling this gets diagnostic output that is invisible at runtime; `MvvmApplication` additionally relies on its own `ReflectionTypeLoadException` catch around `GetExportedValues` for the fatal cases.
- **Case-insensitive ignore matching** uses the raw file name, not the assembly identity, so ignoring "foo.dll" does not ignore a renamed file that loads to the same assembly — and vice versa.
- **`moduleAssemblies` are loaded before the duplicate-`catalog` check adds them to the `assemblies` set:** the set is checked but the assembly is not added to it after a successful module load, so two *different* `moduleAssemblies` entries that resolve to the same file are not caught (only a collision with something cataloged in steps 2–4 is).

### Rough edges

- **`new AssemblyCatalog(moduleAssembly)` in step 5 takes a path string, not the loaded `Assembly`.** `AssemblyCatalog(string)` re-resolves the path and may load a second copy of the module assembly (or throw if the file has moved), whereas steps 2 and 4 correctly pass the `Assembly` instance. A path typo also produces a confusing load error at composition time rather than at the `LoadFrom` line.
- **`ConsoleApp1` passes its own `.exe` filename to the ignore list** — `CreateCompositionContainer(["ConsoleApp1.exe"])` — but the method's ignore matching compares against assembly *file names on disk*, and the entry assembly is added in step 4 regardless; the ignore entry here has no effect and the comment in `ConsoleApp1/Program.cs` ("the ConsoleApp1.exe is not a valid .net assembly") suggests the author was confused about which overload was being called. The overload chosen takes `ignoreAssemblies`, not `moduleAssemblies`.
- **The `catch (Exception)` around per-file loading swallows `InvalidOperationException` from the duplicate check** and everything else (including `FileLoadException`, `BadImageFormatException`), printing only the message. A genuinely broken assembly in the output directory is indistinguishable from a duplicate in any log a developer is likely to see.
- **`Console.WriteLine` as the sole diagnostic channel** is wrong for the WPF host (`MvvmApplication`), where a console is not attached; failures in this method are effectively silent there.