### Purpose

`Common/Reflection/AssemblyHelper.cs` defines `AssemblyHelper`, a static class of small reflection utilities that answer questions about an already-loaded `Assembly`: its file name, its `System.Drawing` metadata attributes (company, product, copyright, trademark, culture, version numbers, public key token), plus two application-level helpers that locate the app's directory and load the application icon from the entry assembly's embedded resources.

It exists so that UI, logging, and composition code can display or log "who am I running as" without each site duplicating the `Attribute.GetCustomAttribute` boilerplate.

### Role in the Codebase

It lives in the `Common` project under `Common/Reflection/`, alongside `Common/Composition/CompositionHelper.cs` — a shared-infrastructure folder the rest of the solution (`Mvvm`, `Mvvm.Wpf`, `Mvvm.Winforms`, the app projects) builds on. It is a helper with no state, no construction, and no project-specific logic; every member is a pure function of the `Assembly` you hand it (or of the executing/entry assembly).

If it were removed, `CompositionHelper` and `MainViewModelBase` would break, and the metadata accessors would have to be re-implemented at every call site.

### Relationships

Dependencies are all framework: `System.Reflection`, `System.IO`, `System.Drawing`, `System.Text`. Nothing in this repository is referenced by `AssemblyHelper` itself.

Callers found in the repository:

- `Common/Composition/CompositionHelper.cs` uses `AssemblyHelper.Extensions` when enumerating `*.dll` and `*.exe` files next to the executing assembly to build its MEF `AggregateCatalog`.
- `Mvvm/MainViewModelBase.cs` calls `AssemblyHelper.LoadApplicationIcon()` in its constructor to set the WPF window's `Icon` from the entry assembly.

The attribute accessors (`GetAssemblyCompany`, `GetAssemblyProduct`, `GetAssemblyCopyright`, `GetAssemblyTrademark`, `GetAssemblyCulture`, `GetAssemblyVersion`, `GetAssemblyFileVersion`, `GetPublicKeyToken`) have no in-repository callers at present; they are a general-purpose API for whoever needs assembly metadata (about `AssemblyHelper` itself, or about assemblies loaded by `CompositionHelper`).

### Behavior & Details

- **`Extensions`** — `public static readonly string[]` containing `".dll"` and `".exe"`. `CompositionHelper` iterates this to gather candidate assembly files; `readonly` means the array reference is fixed, but the array is still publicly mutable, so callers should not treat it as immutable content.

- **`GetAssemblyFileName(assembly)`** — returns `new Uri(assembly.Location).LocalPath`. This normalizes the location into a real file path rather than a `file:///` URI; it is the base for the other two file helpers.

- **`GetApplicationDirectory()`** — the directory containing the *executing* assembly's file. Note the nullable return: if the assembly has no resolvable location, `FileInfo`'s `DirectoryName` can be `null`, which is propagated.

- **`GetAssemblyFileExtension(assembly)`** — file extension of the assembly's location, e.g. `".dll"`.

- **`LoadApplicationIcon()`** — takes the *entry* assembly (`Assembly.GetEntryAssembly()`), not the executing one, and reads the embedded resource named `<entryAssemblySimpleName>.App.ico` via `GetManifestResourceStream`. The name is constructed from `assembly.GetName().Name + ".App.ico"`, so the resource name in your entry assembly must match its simple assembly name exactly (e.g. assembly `MyApp` → resource `MyApp.App.ico`). Returns `null` when there is no entry assembly or no such resource; otherwise returns a new `Icon` (the caller is responsible for disposing it — `MainViewModelBase` uses it synchronously and does not).

- **`GetAssemblyCompany` / `GetAssemblyProduct` / `GetAssemblyCopyright` / `GetAssemblyTrademark` / `GetAssemblyCulture`** — each reads one `Assembly*Attribute` with `Attribute.GetCustomAttribute` and returns the attribute's value, or `string.Empty` when the attribute is absent. The empty-string fallback is the common contract: callers can treat the result as a display string without null checks.

- **`GetAssemblyVersion(assembly)`** — reads `AssemblyVersionAttribute` and returns `attribute.Version`. That property is a `string` holding the `[assembly: AssemblyVersion("x.y.z.w")]` value.

- **`GetAssemblyFileVersion(assembly)`** — same shape for `AssemblyFileVersionAttribute` and its `Version` property.

- **`GetPublicKeyToken(assembly)`** — pulls `assembly.GetName().GetPublicKey()` and formats the whole public key as a lowercase hex string (two hex digits per byte, in order). Returns `string.Empty` if the assembly is not strong-named. Note this is the *public key*, not the key token — it is the long form, not the 16-byte token you typically see in a `.dll`'s identity.

### Rough edges

- `GetAssemblyFileName` assumes `assembly.Location` is a file URI. For dynamically generated or in-memory-only assemblies, `Location` is `string.Empty` and `new Uri("").LocalPath` returns `""`; the downstream `FileInfo("").DirectoryName` would be `null` for `GetApplicationDirectory` and `GetAssemblyFileExtension` would throw. Callers dealing with dynamic assemblies should not use these two.
- `GetAssemblyVersion` and `GetAssemblyFileVersion` return the attribute's declared `Version` string, not the runtime `AssemblyName.Version` — for `AssemblyVersionAttribute` that is the same thing, but the implementation goes through the attribute rather than `assembly.GetName().Version`, so any subtle divergence (e.g. attribute not present but version still in metadata) would surface as `string.Empty`.
- `LoadApplicationIcon` returns a live `Icon` wrapper over an unmanaged GDI+ handle. The repository's only caller (`MainViewModelBase`) never disposes it, so each `MainViewModelBase` instantiation leaks one `Icon` for the lifetime of the process (in practice, one per app launch).