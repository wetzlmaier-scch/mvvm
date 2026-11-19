# Common/Reflection/AssemblyHelper.cs

### Purpose

`AssemblyHelper` is a static class of small reflection utilities that answer questions about an already-loaded `Assembly`: where its backing file lives, what its file metadata attributes say (company, product, version, copyright, and so on), and how to pull the application's icon out of its embedded resources. It exists so that callers elsewhere in the codebase have one place to ask these questions instead of repeating `FileInfo`/`Attribute.GetCustomAttribute` boilerplate against `Assembly` instances.

It does not load assemblies itself. With two exceptions noted below, every method takes an `Assembly` instance and inspects it; the two exceptions are `GetApplicationDirectory`, which inspects the executing assembly, and `LoadApplicationIcon`, which inspects the entry assembly.

### Role in the Codebase

The `Common/Reflection/` directory holds only this file, and the class is a pure leaf: it depends on nothing else in the repository, only on `System.Reflection`, `System.IO`, `System.Text`, and `System.Drawing`. Callers depend on it, never the reverse.

The most load-bearing consumer is `Common/Composition/CompositionHelper.cs`, whose `CreateCompositionContainer` builds a MEF `AggregateCatalog` by enumerating files next to the executing assembly. It iterates `AssemblyHelper.Extensions` to collect both `.dll` and `.exe` files, so that constant defines which files the MEF catalog picks up.

The other consumer is `Mvvm/MainViewModelBase.cs`, whose constructor calls `LoadApplicationIcon()` (and then hands the icon to `BitmapSourceHelper`) to set the WPF window icon.

If the class were removed, the MEF composition bootstrap would lose its list of file extensions to enumerate, and the main view model would lose its icon source. The attribute-reading methods have no other visible consumers, but they are the public API of the class and may be used by hosts built on this shared code.

### Relationships

- **Imports:** nothing from within the repository — `System.Drawing` (`Icon`), `System.IO`, `System.Reflection`, and `System.Text` only.
- **`Common/Composition/CompositionHelper.cs`** — reads `AssemblyHelper.Extensions` while scanning for assemblies to add to the MEF catalog.
- **`Mvvm/MainViewModelBase.cs`** — calls `LoadApplicationIcon()` in its constructor to obtain the window icon.

### Behavior & Details

All members are `static` on a `static` class; there is no state to worry about.

**`Extensions`** — `public static readonly string[]` holding `{ ".dll", ".exe" }`. It is the single source of truth for which file types count as loadable assemblies, and `CompositionHelper` relies on it when enumerating the output directory.

**`GetAssemblyFileName(Assembly)`** — returns the local file path of the assembly by running `new Uri(assembly.Location).LocalPath`. The `Uri` round-trip normalizes the path (e.g. converts UNC-style `file://` locations to a plain path). If `assembly.Location` is not a file path, the `Uri` constructor will throw.

**`GetApplicationDirectory()`** — returns the directory containing the *executing* assembly (`Assembly.GetExecutingAssembly()`), i.e. the directory of the assembly this very code ships in. Note this is not necessarily the entry assembly's directory: for a WPF app it is the folder of the assembly containing this class, which in a typical layout is the application's output directory. Returns `null` when the location has no directory component.

**`GetAssemblyFileExtension(Assembly)`** — the extension of the assembly's backing file (from the same normalized path as above), e.g. `".exe"`.

**`LoadApplicationIcon()`** — returns the application's icon, or `null`. It takes `Assembly.GetEntryAssembly()`, and if that is `null` (e.g. when the code is running under the design-time host or a test runner without a proper entry point) it returns `null`. Otherwise it opens the embedded manifest resource named `"<assembly simple name>.App.ico"` — so an assembly called `MyApp` must embed an icon resource named `MyApp.App.ico`. If that resource is absent, it again returns `null`. The caller is expected to handle `null` (as `MainViewModelBase` does); there is no fallback icon here.

**Attribute readers** — `GetAssemblyCompany`, `GetAssemblyProduct`, `GetAssemblyCopyright`, `GetAssemblyTrademark`, `GetAssemblyCulture`, `GetAssemblyVersion`, and `GetAssemblyFileVersion` all follow one shape: they read the matching `Assembly*Attribute` via `Attribute.GetCustomAttribute`, and return the attribute's value, or `string.Empty` when the attribute is not present on the assembly. So these never return `null` — absence of metadata is indistinguishable from an empty-string value unless you check for `string.Empty`.

**`GetPublicKeyToken(Assembly)`** — returns the assembly's public key token as a lowercase hex string, e.g. `"b77a5c561934e089"`. It reads `assembly.GetName().GetPublicKey()`; if the assembly is unsigned (token is `null`), it returns `string.Empty`. Note the name says "token" but the body hex-encodes the full **public key**, not the 8-byte token — callers expecting the short token form will get a much longer string.

### Rough edges

- `GetPublicKeyToken` is named after the token but actually serializes the full public key bytes, so its output is not the conventional public-key token.
- `LoadApplicationIcon` hard-codes the manifest resource name as `<simple name>.App.ico`; an assembly whose icon resource uses any other convention will silently get `null`.