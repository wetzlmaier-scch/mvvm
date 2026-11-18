# Common/Composition/CompositionHelper.cs

### Purpose

`CompositionHelper` is a static helper that builds a MEF (`System.ComponentModel.Composition`) `CompositionContainer` from all loadable assemblies next to the running application. It exists so that the application's modules can be discovered and wired up automatically at startup: instead of registering each export/import by hand, the container is populated from every `.dll` and `.exe` in the application directory (or its subdirectories), and the app then pulls out whatever it needs — in practice the `IModuleController` exports — from it.

### Role in the Codebase

This class is the composition bootstrap for the solution. It lives in `Common` and is consumed by the host applications:

- `Mvvm.Wpf/MvvmApplication.cs` calls `CreateCompositionContainer(moduleAssemblies, ignoreAssemblies, searchOption)` in `OnStartup`, then resolves `IModuleController` exports, calls `Initialize()` and `Run()` on them, and disposes the container in `OnExit`.
- `WinFormsApp1/Program.cs` calls the no-argument overload in `Main` and follows the same module-controller pattern.
- `ConsoleApp1/Program.cs` calls it with an ignore list and resolves a specific view-model export (`IUserViewModel`) directly.

If it were removed, the hosts would have no way to assemble the container from the directory of assemblies, and module discovery would have to be replaced by explicit registration.

### Relationships

- Depends on `Common.Reflection.AssemblyHelper` (in `Common/Reflection/AssemblyHelper.cs`), used only for its `Extensions` constant, the `string[] { ".dll", ".exe" }` that determines which files in the directory are considered candidate assemblies.
- Everything else is from `System.ComponentModel.Composition.Hosting` (external, .NET's MEF): `AggregateCatalog`, `AssemblyCatalog`, `CompositionContainer`, `CompositionBatch`.
- The returned `CompositionContainer` is what all hosts use for `GetExportedValues<...>`; callers treat it as disposable (both hosts dispose it on shutdown).

### Behavior & Details

There are two parameterless/thin overloads that just delegate to the real implementation `CreateCompositionContainer(string[] moduleAssemblies, string[] ignoreAssemblies, SearchOption searchOption)`. All three share one behavior; the overloads only let callers skip the list(s) they don't need.

The full overload works in four stages:

1. **Collect candidate files.** It takes the directory of `Assembly.GetExecutingAssembly()` (i.e. the directory containing the `Common` assembly, which in practice is the application's output directory) and enumerates files matching `*` + each extension in `AssemblyHelper.Extensions` (`*.dll`, `*.exe`) using the given `searchOption` (`TopDirectoryOnly` by default, or `AllDirectories` if a caller wants subdirectories).

2. **Load and catalog each file.** For every file not in `ignoreAssemblies` (matched case-insensitively by file name), it calls `Assembly.LoadFrom`. Two bookkeeping rules apply:
   - If the loaded assembly's location is already in the catalog set, it throws `InvalidOperationException` ("Assembly ... is already loaded") — this throw is inside the per-file `try`, so it is caught and only printed to the console; the loop then moves on.
   - Otherwise it adds an `AssemblyCatalog` to the `AggregateCatalog` and records the assembly's location (lower-cased) in a `HashSet`.
   - Any other exception (a corrupt file, a bad assembly such as the `ConsoleApp1.exe` case in the sample, etc.) is also caught and its message is written to `Console.WriteLine` — a failing assembly is silently skipped from the container rather than aborting startup.

3. **Guarantee the Common and entry assemblies are present.** After the file scan, it adds the executing assembly and, if different, the entry assembly (`Assembly.GetEntryAssembly()`) to the catalog, again only if their locations aren't already recorded. This matters because the scan depends on the `Common` assembly sitting next to the executable, and the entry assembly's location can in edge cases not be one of the enumerated files.

4. **Load module assemblies and self-export the container.** For each name in `moduleAssemblies` it loads the assembly with `Assembly.LoadFrom` and adds it to the catalog. Note this stage is *not* in a try/catch: an unloadable module assembly, or one already present, throws `InvalidOperationException` and propagates to the caller. Finally it creates the `CompositionContainer`, adds the container itself as an exported value via a `CompositionBatch` (so other components can import the container), composes the batch, and returns the container.

A caveat worth knowing: `moduleAssemblies` entries are loaded with `Assembly.LoadFrom` but the catalog entry is built from the *name string*, not the loaded `Assembly` object (unlike the file-scan stage). The comment in the code refers to App.config as the source for module assemblies, but in the current code no caller actually passes a non-empty `moduleAssemblies` — `WpfApp1.App` inherits `MvvmApplication`'s parameterless constructor, so both arrays are empty in practice.

### Rough edges

- Failures in the file-scan loop are only written to the console (`Console.WriteLine`), which in a WPF app goes nowhere a user sees; a missing or broken module assembly can make exports silently absent until a later `GetExportedValues` fails or finds nothing.
- The "already loaded" `InvalidOperationException` thrown inside the `try` is swallowed by its own `catch`, so a genuine duplicate-assembly problem produces just a console line.