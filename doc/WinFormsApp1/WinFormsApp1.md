# WinFormsApp1.csproj

**Path:** `WinFormsApp1/WinFormsApp1.csproj`

### Purpose

This is the MSBuild project file for `WinFormsApp1`, the WinForms desktop host application in the solution. It declares the project as a Windows executable targeting `net10.0-windows` with Windows Forms and nullable reference types enabled, and wires it up to the three projects that supply its runtime behaviour. It contains no application logic itself.

### Role in the Codebase

It is the build definition for the application's entry point. The sibling source file `Program.cs` is what this project compiles: `Main()` uses `CompositionHelper.CreateCompositionContainer()` from `Common` to build an MEF composition container, then discovers all exported `IModuleController` instances, calls `Initialize()` on each followed by `Run()`, and returns `ExitCodes.Ok` (or `ExitCodes.Error` if no module is found). The `OutputType` of `WinExe` is what turns that code into a windowed desktop executable rather than a library.

Without this file, the `WinFormsApp1` namespace and its `Program` class cannot be built, and the application has no host for its modules.

### Relationships

The three project references determine what lands in the output directory, and the directory matters: `CompositionHelper.CreateCompositionContainer()` (in `Common/Composition/CompositionHelper.cs`) scans the directory containing the executing assembly and loads every assembly it finds there as an MEF catalog. So the referenced assemblies are not just compile-time dependencies — the runtime discovery of modules depends on them being copied to the output folder.

- `..\Common\Common.csproj` — provides `CompositionHelper`, `ExitCodes`, and the composition infrastructure `Program.cs` uses.
- `..\Mvvm.Winforms\Mvvm.Winforms.csproj` — the WinForms flavour of the MVVM layer (it itself references `..\Mvvm\Mvvm.csproj`).
- `..\UserManagement\UserManagement.csproj` — a feature module; it pulls in `BusinessLayer`, `DataAccessLayer.Hibernate`, and `Logging` transitively, which is how data access and logging end up in the WinForms app's deployment.

### Behavior & Details

- `OutputType` is `WinExe`, producing a Windows GUI executable with no console window.
- `TargetFramework` is `net10.0-windows`, matching the other projects in the solution, so the whole tree builds with the same runtime.
- `UseWindowsForms`, `Nullable`, and `ImplicitUsings` are all enabled here; the `System.ComponentModel.Composition` package used by `Program.cs` is not referenced directly in this project but comes transitively via `Common` and `UserManagement`.
- There are no explicit package references or other item groups; everything the app needs at build time flows through the three project references.