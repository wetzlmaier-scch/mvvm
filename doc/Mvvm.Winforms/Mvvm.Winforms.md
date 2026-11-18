# Mvvm.Winforms.csproj

## Purpose

This is the project file (build definition) for the `Mvvm.Winforms` class library. It configures how the WinForms flavor of the MVVM layer compiles: it targets `net10.0-windows`, enables the Windows Forms runtime (`UseWindowsForms`), and enables nullable references and implicit usings.

The project contains exactly one source file, `MessageService.cs`, a WinForms implementation of the `IMessageService` interface from the `Mvvm` project — so this csproj is what makes that single type buildable and consumable as a library.

## Role in the Codebase

- The only build configuration for `Mvvm.Winforms`; without it the folder is just a directory of C# files with no assembly to compile into.
- By setting `UseWindowsForms=true`, it gives `MessageService.cs` access to `MessageBox` and the related WinForms types it uses.
- Declares the sole project dependency: `..\Mvvm\Mvvm.csproj`. That project provides the `IMessageService` interface (and, transitively, `Common`), so this project is a thin WinForms adapter over the framework-agnostic MVVM core.
- It is referenced as a `ProjectReference` by host applications that want MVVM on WinForms (e.g. `WinFormsApp1` in the solution).

## Relationships

- Depends on the `Mvvm` project (`..\Mvvm\Mvvm.csproj`), which defines `IMessageService` and the MVVM infrastructure (`CommunityToolkit.Mvvm`, `CommunityToolkit.Diagnostics`, `FluentValidation`) that the consuming side uses.
- No NuGet package references of its own — everything comes transitively through the `Mvvm` project.

## Behavior & Details

- `TargetFramework` is `net10.0-windows`, matching the sibling `Mvvm` project, so both compile for the same platform.
- `ImplicitUsings` and `Nullable` are enabled, matching the rest of the solution.
- `UseWindowsForms` is the only property that distinguishes it from a plain class library — it pulls in the WinForms assemblies.