# WpfApp1/View/UserView.xaml.cs

### Purpose

This file is the code-behind half of `UserView`, the main window of the WPF host project (`WpfApp1`). Its entire job is to make the window a MEF export: the `[Export(typeof(IMainView<bool?, double>))]` attribute registers `UserView` in the composition container so the module controller can import it. All actual UI lives in the paired `UserView.xaml`; the constructor just calls `InitializeComponent()`.

### Role in the Codebase

`WpfApp1` is one of several host projects in the repository (a WinForms sibling does the same with `UserForm`). In this project, `UserView` is the application's main window and the sole exporter of `IMainView<bool?, double>`. `DefaultModuleController` (`WpfApp1/Controller/DefaultModuleController.cs`) imports that contract; if this export were removed, MEF composition of the controller would fail and the app would not start.

The class is `partial` — the other half is generated from `UserView.xaml`, which declares the window's layout: a "Current Users" list bound to `ShowUsersViewModel.Users`, and an "Add New User" form bound to `AddUserViewModel`, plus a `Loaded` trigger that fires `ShowUsersViewModel.LoadedCommand`. Nothing here overrides or implements any of the interface members directly.

### Relationships

- **`Window`** (WPF, `System.Windows`) — base class. This is where the members required by the interface come from: `Show()`, `Close()`, the `Closing`/`Closed` events, `DataContext` (from `IView`), and the `Left`/`Top`/`Width`/`Height` properties. Because `TDimension` is `double` and WPF exposes those properties as `double`, `Window` satisfies `IMainView<bool?, double>` without any code written in this file.
- **`IMainView<TDialogResult, TDimension>`** — defined in `Mvvm/IMainView.cs`; extends `IDialogView<TDialogResult>` (`Mvvm/IDialogView.cs`), which extends `IView` (`Mvvm/IView.cs`). Note the dialog-result type is `bool?` rather than WinForms' `DialogResult`, and the dimension type is `double` rather than `int` — a WPF-appropriate pairing.
- **`DefaultModuleController`** — the only consumer. Its `[ImportingConstructor]` receives `IMainView<bool?, double>` and `IMainViewModel`, sets `MainView.DataContext = mainViewModel` (so the `UserViewModel` from the `UserManagement` project drives the XAML bindings), then calls `MainView.Show()` in `Run()` and `MainView.Close()` in `Shutdown()`.

### Behavior & Details

The logic is minimal by design:

- The class declares no methods, properties, or event handlers of its own.
- `UserView()` calls `InitializeComponent()`, which instantiates the window from the XAML markup.
- All interface members are supplied by the inherited `Window` implementation, so no member requires an override here.
- The export attribute is what couples the class into the application; without it the window would exist but never be composed or shown.

One caveat worth knowing: `UserView` does not set `DialogResult` and no caller reads it. The `bool?` type parameter is part of the MEF contract's identity (it must match what `DefaultModuleController` imports) but carries no actual dialog-result semantics in the WPF host.