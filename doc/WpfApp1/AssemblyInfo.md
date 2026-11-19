# `WpfApp1/AssemblyInfo.cs`

### Purpose

This is the assembly-level attributes file for the WPF application project (`WpfApp1`). It exists because WPF requires the `[assembly: ThemeInfo]` attribute to be present on the application's assembly so that the framework knows where to look for resource dictionaries during theme resolution. Without it, WPF resource lookup for the application assembly is misconfigured.

### Role in the Codebase

It is a standard, compiler-convention file sitting alongside `WpfApp1/App.xaml` and `WpfApp1/App.xaml.cs` — the entry point of the desktop WPF application (an `MvvmApplication` derived from the `Mvvm.Wpf` project). Like the other source files in `WpfApp1`, it compiles into the single `WpfApp1` executable. Removing it would not cause a compile error, but WPF's resource dictionary lookup for this assembly would fall back to defaults instead of the explicitly declared locations.

### Relationships

- It depends only on `System.Windows` (framework type `ResourceDictionaryLocation` from the WPF libraries, enabled by `<UseWPF>true</UseWPF>` in `WpfApp1.csproj`).
- It is consumed indirectly by the WPF framework at runtime: the framework reads this attribute (via reflection) when resolving resources for any `ResourceDictionary` referenced from the assembly — including those in `App.xaml` and `WpfApp1/View/UserView.xaml`.
- Nothing in the codebase explicitly references or calls into this file; it is metadata, not code with callers.

### Behavior & Details

The single attribute declared here is `[ThemeInfo]` with two arguments, each a `ResourceDictionaryLocation` enum value that points WPF's resource lookup chain at a location:

- **First argument — `ResourceDictionaryLocation.None`**: theme-specific resource dictionaries (e.g., `Themes/ aero/Generic.xaml`) are declared to live in *no* separate location. If a resource is not found in a page's or the application's resource dictionaries, WPF will not consult a theme dictionary from this assembly.
- **Second argument — `ResourceDictionaryLocation.SourceAssembly`**: the generic resource dictionary (typically `Themes/Generic.xaml`) lives in *this* assembly. If a resource is not found in the page, application, or theme dictionaries, WPF falls back to the generic dictionary bundled with the `WpfApp1` assembly itself.

This is the default pair the Visual Studio WPF project template generates. The project currently does not contain a `Themes/` folder, so in practice no theme or generic dictionaries exist to look up — the attribute records the intended lookup policy for when such dictionaries are added.

### Rough edges

This file duplicates, in part, what the SDK-style project can infer automatically; in modern WPF projects it is often the template's leftover convention rather than something deliberately configured. No `Themes/` folder exists in `WpfApp1`, so both lookup locations currently point at nothing — the attribute is declarative intent rather than active configuration.