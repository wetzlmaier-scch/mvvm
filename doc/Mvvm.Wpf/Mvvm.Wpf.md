### Purpose
`Mvvm.Wpf/Mvvm.Wpf.csproj` is the MSBuild project file for the `Mvvm.Wpf` library — the WPF-specific half of this repository's Mvvm framework. It defines how the project's two classes, `MvvmApplication` and `MessageService`, are compiled. The project exists to make the framework's module-based, MEF-composed application shell available for WPF, complementing the sibling `Mvvm` core project and the `Mvvm.Winforms` / `Mvvm.Console` variants.

### Role in the Codebase
It is one of several sibling projects in the solution (`Demo.slnx`), each providing a UI-platform binding for the shared `Mvvm` framework. This file declares the two things that make the WPF binding work:

- `<UseWPF>true</UseWPF>` — pulls in WPF assemblies so `MvvmApplication` can derive from `System.Windows.Application` and `MessageService` can use `MessageBox`.
- `TargetFramework` `net10.0-windows` — WPF requires the Windows desktop runtime.

If this file were removed, `MvvmApplication` and `MessageService` could no longer be built or referenced by a WPF host application such as `WpfApp1`.

### Relationships
- References `..\Mvvm\Mvvm.csproj` — the framework core. Both projects pin `System.ComponentModel.Composition` 10.0.6 (the MEF package), which `MvvmApplication` uses to build its `CompositionContainer` and discover `IModuleController` exports.
- Consumers are WPF applications that instantiate `MvvmApplication` as their `Application` subclass (for example, the `WpfApp1` project in this solution), so module assemblies and message-box UI flow through the WPF stack instead of WinForms or the console.

### Behavior & Details
There is no build-time logic beyond standard SDK-style properties: implicit usings and nullable reference types are enabled, matching the sibling `Mvvm` project. No custom targets, analyzers, or output settings are defined, so the build inherits everything from the .NET SDK.