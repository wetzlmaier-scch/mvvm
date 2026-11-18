# WpfApp1/AssemblyInfo.cs

### Purpose

This file carries the WPF `ThemeInfo` assembly attribute for the `WpfApp1` desktop application. It tells the WPF resource lookup engine where to find theme-specific and generic resource dictionaries for this assembly, which is what lets controls and resources resolve `ThemeDictionary` and `GenericDictionary` correctly at runtime.

### Role in the Codebase

`WpfApp1` is the WPF executable project of the solution (see `WpfApp1/WpfApp1.csproj`: `OutputType` `WinExe`, `UseWPF` enabled, `net10.0-windows`). This file is the standard, template-generated `AssemblyInfo.cs` that WPF project scaffolding emits; it is not hand-maintained application code. The application itself is declared in `WpfApp1/App.xaml` (`WpfApp1.App`, derived from `Mvvm.Wpf`'s `MvvmApplication`), and its behavior lives elsewhere in the project and its referenced `Common`, `Mvvm.Wpf`, and `UserManagement` projects.

If the file were removed, nothing in application logic breaks — there are no application resources here — but WPF would fall back to default dictionary lookup locations, which matters only if theme or generic resource dictionaries are ever added to the assembly.

### Relationships

- Depends on `System.Windows` (the WPF framework), specifically the `ThemeInfo` attribute and `ResourceDictionaryLocation` enum from the framework, not from this repository.
- Consumed implicitly by the WPF runtime and by anything in `WpfApp1` that relies on resource dictionary lookup. It has no callers in the repository's own code; it is pure metadata compiled into the assembly.

### Behavior & Details

- `[assembly: ThemeInfo]` sets two lookup locations for this assembly:
  - First argument `ResourceDictionaryLocation.None` — no theme-specific resource dictionary location is declared. Theme dictionaries are therefore not searched for in this assembly, per the standard fallback chain (page → application → theme).
  - Second argument `ResourceDictionaryLocation.SourceAssembly` — the generic resource dictionary, if one exists, is expected in the `Themes/Generic.xaml` dictionary of this assembly itself (the default WPF convention).
- The attribute has no runtime side effects beyond registering this metadata on the assembly; no classes, methods, or state are defined here.