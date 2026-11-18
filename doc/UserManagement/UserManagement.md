# UserManagement.csproj

`UserManagement/UserManagement.csproj`

## Purpose

This file is the SDK-style project file for the `UserManagement` library. It is build configuration, not application code: it declares the target framework, nullable/implicit-usings settings, and the dependencies needed to compile the view models, models, and messages in the `UserManagement` directory. The project itself holds the view-model layer for user management (`ViewModel/`, `Model/`, `Messaging/`, `Design/`), written UI-framework-agnostic so several host apps can consume it.

## Role in the Codebase

It is a class library — there is no `OutputType` of `Exe` or `WinExe`, so it compiles to a DLL rather than an executable. It is not an entry point. Removing it would break every project that references it: `WpfApp1`, `WinFormsApp1`, `ConsoleApp1`, and `TestProject1` all declare a `ProjectReference` to `..\UserManagement\UserManagement.csproj`, so their builds would fail to resolve the `UserManagement` namespace.

## Relationships

- **Project references (dependencies):**
  - `..\BusinessLayer\BusinessLayer.csproj` — supplies `IUserService`, which the view models import via MEF (`[Import] public IUserService UserService`) and call, e.g. `UserService.GetActiveUsers()` in `ShowUsersViewModel`.
  - `..\DataAccessLayer.Hibernate\DataAccessLayer.Hibernate.csproj` — transitively provides the `Entity.Hibernate` entity types that `UserManagement` code references.
  - `..\Logging\Logging.csproj` — supplies `ILoggingService`, likewise consumed via `[Import]`.
  - `..\Mvvm\Mvvm.csproj` — supplies `ViewModelBase`, the base class of the view models in this project.
- **Package reference:** `System.ComponentModel.Composition` 10.0.6 — the MEF implementation used for `[Export]`/`[Import]` composition of the view models.
- **Consumers:** the four host/test projects listed above compile this DLL into their apps; the WPF and WinForms hosts pair it with their own `Mvvm.Wpf`/`Mvvm.Winforms` projects to render the view models.

## Behavior & Details

- `TargetFramework` is `net10.0-windows`, which is what makes this library consumable only by Windows-targeted apps — matching the Windows-only hosts that reference it.
- `ImplicitUsings` and `Nullable` are both enabled, so the C# sources in the project rely on compiler-inferred usings and nullable reference type checks.
- The `System.ComponentModel.Composition` package is required because .NET no longer ships MEF in the base class libraries; without this reference the `[Export]`/`[Import]` attributes used throughout `ViewModel/` would not resolve.
- There are no custom build targets, no `OutputType`, and no content items — the project relies entirely on default SDK compilation of the `.cs` files in its directory tree.

### Rough edges

- The project references `DataAccessLayer.Hibernate` even though its own sources import `Entity.Hibernate` types; the reference appears to exist to make those entity types available, which couples this view-model layer to the persistence stack transitively.