# WpfApp1/View/UserView.xaml.cs

### Purpose

This code-behind file is the C# half of the `UserView` window — the main "User Management" window of the WPF application. It does almost nothing on its own: it registers the window as an MEF export so the application's controller can find, show, and close it without ever naming the concrete type. The actual UI layout, bindings, and styling live in the paired `UserView.xaml` file.

### Role in the Codebase

`UserView` is the presentation entry point of the WPF app. The startup flow is:

1. `App` (`WpfApp1/App.xaml.cs`, derived from `MvvmApplication`) boots MEF composition.
2. `DefaultModuleController` (`WpfApp1/Controller/DefaultModuleController.cs`) is the exported `IModuleController`. Its importing constructor receives an `IMainView<bool?, double>` and an `IMainViewModel`; it assigns the view model to the view's `DataContext` and calls `MainView.Show()` in `Run()`.
3. `UserView` is the only `IMainView<bool?, double>` export in the WPF project (the WinForms app has its own parallel `UserForm` implementing `IMainView<DialogResult, int>`), so this is the class MEF hands to the controller.

If the `[Export]` attribute or the interface implementation were removed, MEF would fail to satisfy the controller's constructor and the application would not launch.

### Relationships

- **`IMainView<bool?, double>`** (`Mvvm/IMainView.cs`): the contract this class implements. It inherits `IDialogView<bool?>` (`Mvvm/IDialogView.cs`) and `IView` (`Mvvm/IView.cs`). Concretely, `UserView` fulfils: `DataContext` (from `IView`, satisfied by `Window`), `Show()`/`Close()`/`Closing`/`Closed` (inherited from `Window`), `DialogResult` and `Left`/`Top`/`Width`/`Height` (satisfied by `Window` properties typed compatibly with `bool?` and `double`). Nothing here overrides any of that — it all comes for free from `Window`.
- **`[Export]`** (`System.ComponentModel.Composition`): makes `UserView` discoverable to the MEF container used by the `Mvvm` framework.
- **`UserView.xaml`**: the markup counterpart. The `x:Class="WpfApp1.View.UserView"` attribute in the XAML and the `partial` keyword here join them into one class; the XAML's data bindings (e.g. `ShowUsersViewModel.Users`, `AddUserViewModel.SaveUserCommand`, the `Loaded` event trigger that invokes `LoadedCommand`) are all resolved against whatever `DefaultModuleController` assigns as `DataContext` — the `IMainViewModel` export, `UserViewModel` (`UserManagement/ViewModel/UserViewModel.cs`).

The only caller in this repository is `DefaultModuleController`, which depends on the interface, not the class.

### Behavior & Details

- The class is `partial`; the other part (generated from `UserView.xaml` at build time) contains the `InitializeComponent()` method that the constructor calls to wire up the visual tree. The constructor performs no other work.
- The generic arguments encode how this app's dialog/position contract is typed: `bool?` for the dialog result, `double` for the view's position and size — matching WPF's `Nullable<bool> Window.DialogResult` and `double`-based `Left`/`Top`/`Width`/`Height`. The WinForms twin uses `DialogResult`/`int` instead, which is why the two apps need separate exports rather than sharing one.
- Because no member is implemented in this file at all, behaviour changes almost never require touching it. Only the export type or the interface conformance would be reasons to edit it.

### Rough edges

- `IMainView<TDialogResult, TDimension>` is a weakly typed contract: the "dimension" parameters (`Left`, `Top`, `Width`, `Height`) are all the same type, and the dialog result is an unconstrained generic. This forces each host app to pick an arbitrary type pairing (`bool?`/`double` here, `DialogResult`/`int` in WinForms), which is why the exports cannot be shared across the two projects.