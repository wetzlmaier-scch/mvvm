# `Mvvm/Mvvm.csproj`

### Purpose

The project file for the `Mvvm` library — the shared, framework-agnostic core of the repository's MVVM stack. It configures the project so that the abstract base view models and view/view-model contracts in the `Mvvm` folder (`ViewModelBase`, `DisposableViewModelBase`, `ValidatableViewModelBase`, `IViewModel`, `IMainViewModel`, `IDialogView`, and their companions) compile against the third-party MVVM and validation libraries those types depend on. It declares the target framework, the NuGet packages, and the one project reference; it contains no logic itself.

### Role in the Codebase

This is the build definition for the middle layer of a three-part MVVM stack:

- `Mvvm` (this project): abstract base classes and interfaces shared by all UI frameworks.
- `Mvvm.Wpf`, `Mvvm.Winforms`, `Mvvm.Console`: framework-specific implementations, each of which references `Mvvm.csproj` directly.
- The application projects (`WpfApp1`, `WinFormsApp1`, `ConsoleApp1`) and `UserManagement` also reference it (through the stack above or directly).

If this project file were removed, none of the `Mvvm` source files could build, and every project that depends on the `Mvvm` namespace would fail to compile.

### Relationships

- **`Common` project** (`..\Common\Common.csproj`): the only project reference. `ViewModelBase.cs` uses `ExtendedGuard` from `Common.Diagnostics`, so this library transitively depends on `Common`.
- **NuGet packages**, each tied to specific sources in the folder:
  - `CommunityToolkit.Mvvm` — `ViewModelBase` extends `ObservableObject` from it.
  - `CommunityToolkit.Diagnostics` — `ViewModelBase` and `DisposableViewModelBase` use its `Guard` methods.
  - `FluentValidation` — `ValidatableViewModelBase<T>` takes an `IValidator<T>` and reports errors through `INotifyDataErrorInfo`.
  - `System.ComponentModel.Composition` — not consumed by any file in `Mvvm` itself, but by the sibling projects `Mvvm.Wpf`, `Mvvm.Winforms`, and `Mvvm.Console` (e.g. `MvvmApplication.cs` and the `MessageService` classes). Declaring it here keeps the MEF package reference available across the stack.

### Behavior & Details

- **Target framework is `net10.0-windows`.** The `-windows` TFM is required because `ViewModelBase` references `System.Windows` types (`FrameworkElement`, `DesignerProperties`) to detect design mode — so this project cannot build for a plain `net10.0` target, even though the rest of the code is UI-framework-agnostic.
- **`Nullable` is enabled** (`<Nullable>enable</Nullable>`), and **implicit usings** are on; source files in the folder rely on both.
- **No output-type override**, so it builds as a class library — the default for `Microsoft.NET.Sdk`.

### Rough edges

- The `System.ComponentModel.Composition` package reference lives in this project's file, but no file in the `Mvvm` folder uses that namespace; it is effectively declared here on behalf of the `Mvvm.Wpf`/`Mvvm.Winforms`/`Mvvm.Console` projects, which use it directly.