# Common/Reflection/AssemblyHelper.cs

### Purpose

`AssemblyHelper` is a static class of small reflection utilities that answer questions about an already-loaded `Assembly`: where its file is, what its attributes say (company, product, version, copyright, etc.), and how to load the application's embedded icon. It exists so callers do not repeat `Attribute.GetCustomAttribute` lookups, `Uri`/`FileInfo` path parsing, and manifest-resource lookup boilerplate scattered through the app.

### Role in the Codebase

It lives in `Common/Reflection/`, the sole file in that directory, and is part of the `Common` assembly. It is a pure helper with no internal dependencies: it references only framework namespaces (`System.Reflection`, `System.IO`, `System.Drawing`, `System.Text`). Nothing it provides is infrastructure — it is application metadata plumbing. Removing it would break two call sites: `CompositionHelper` (which needs `Extensions`) and `MainViewModelBase` (which needs `LoadApplicationIcon`). The attribute accessors are, as far as this repository shows, available but not currently called from within the repo.

### Relationships

- **`Common/Composition/CompositionHelper.cs`** — the main internal consumer. Its `CreateCompositionContainer` enumerates files next to `Assembly.GetExecutingAssembly()` and uses `AssemblyHelper.Extensions` (`.dll`, `.exe`) to decide which files are candidate assemblies for the MEF `AggregateCatalog`.
- **`Mvvm/MainViewModelBase.cs`** — its constructor calls `LoadApplicationIcon()` and converts the resulting `Icon` into a WPF `ImageSource` to set the window icon.
- Callers pass in `Assembly` instances; the class itself never loads assemblies. `LoadApplicationIcon` is the exception in that it fetches the *entry* assembly (`Assembly.GetEntryAssembly()`) itself.

### Behavior & Details

**File locations**

- `GetAssemblyFileName(assembly)` — returns `new Uri(assembly.Location).LocalPath`, i.e. the file system path of the assembly, decoded from its location URI. This is the basis for the next two methods.
- `GetApplicationDirectory()` — the directory containing the *executing* assembly's file (nullable; the directory of a file path). This is the de facto "application directory" used elsewhere for discovery.
- `GetAssemblyFileExtension(assembly)` — the file extension (`.dll` / `.exe`) of the assembly's file, via `FileInfo`.

**Application icon**

`LoadApplicationIcon()` targets the *entry* assembly, not the executing one, and looks for an embedded manifest resource named `<entryAssemblySimpleName>.App.ico` (e.g. `MyApp.App.ico` for an assembly named `MyApp`). It returns `null` if there is no entry assembly (e.g. running under a host) or if the resource is absent. The `Icon` is wrapped around a live `Stream` from the manifest resource; note the caller (`MainViewModelBase`) uses it once to build a bitmap and does not dispose it.

**Attribute accessors**

`GetAssemblyCompany`, `GetAssemblyProduct`, `GetAssemblyCopyright`, `GetAssemblyTrademark`, `GetAssemblyCulture`, `GetAssemblyVersion`, and `GetAssemblyFileVersion` all follow one pattern: `Attribute.GetCustomAttribute(assembly, typeof(XxxAttribute))` cast to nullable, returning the attribute's property or `string.Empty` when the attribute is absent. They never throw for a missing attribute — missing metadata yields an empty string, not `null`.

**Public key token**

`GetPublicKeyToken(assembly)` reads `assembly.GetName().GetPublicKey()` and hex-encodes each byte as two lowercase hex digits (`{0:x2}`). Returns `string.Empty` for unsigned assemblies (null public key). Note this formats the full public *key*, not the short 16-byte public key token that appears in a strong-name string.

### Rough edges

- `GetAssemblyFileName` calls `new Uri(assembly.Location)` and will throw `UriFormatException` for dynamic/in-memory assemblies whose `Location` is empty; `GetApplicationDirectory` and `GetAssemblyFileExtension` inherit that behavior.
- The name `GetPublicKeyToken` is misleading: it serializes the full public key (`GetPublicKey()`), not the public key token (the lower 8 bytes of the SHA-1 hash of the key) that `GetName()` exposes via token-based APIs.