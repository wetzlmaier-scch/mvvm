# WinFormsApp1/View/UserForm.Designer.cs

The `UserForm` form is the WinForms main window of the user-management application, pairing a designer-generated layout with hand-written MVVM code-behind:

- `WinFormsApp1/View/UserForm.Designer.cs` — designer-generated partial class containing the control declarations and the layout produced by `InitializeComponent()`.
- `WinFormsApp1/View/UserForm.cs` — the hand-written partial class: the form's contract (`IMainView<DialogResult, int>`), the MEF export, and all view-to-view-model binding logic.
- `WinFormsApp1/View/UserForm.resx` — standard WinForms resource file; it holds no resources beyond the `errorProvider.TrayLocation` metadata that the designer emits for its component tray.

The three partials compile into a single `UserForm` type.

### Purpose

This form exists to give the `UserManagement` module a native WinForms surface: show the current users and accept a new user. It is one of two platform views of the same view models — the WPF project (`WpfApp1/View/UserView`) binds to the same view-model commands and properties declaratively, while this WinForms form wires everything up imperatively with `DataBindings`, event handlers, and manual `BindingSource` refreshes.

### Role in the Codebase

`UserForm` is the entry window of the WinForms host. `Program.Main` builds a MEF composition container and runs every exported `IModuleController`; `DefaultModuleController` (in `WinFormsApp1/Controller/DefaultModuleController.cs`) takes `IMainView<DialogResult, int>` and `IMainViewModel` as constructor imports and calls `Application.Run((Form)MainView)`. Because `UserForm` is the only type in the WinForms app exporting `IMainView<DialogResult, int>`, it is what gets composed and shown. Removing or de-exporting this form would leave the WinForms app with no main view to run, and there is no other WinForms view to fall back to.

### Relationships

- `Mvvm` (shared, project-local) — the form implements `IMainView<DialogResult, int>` (`Mvvm/IMainView.cs`, which extends `IDialogView<T>` and adds `Left`/`Top`/`Width`/`Height`); `Form` provides the dialog members (`Show`, `Close`, `DialogResult`, `Closing`/`Closed`). The overridable `DataContext` property comes from `Mvvm/IView.cs`.
- `UserManagement.ViewModel` — `DataContext` is cast to `IUserViewModel`, whose `UserViewModel` implementation (`UserManagement/ViewModel/UserViewModel.cs`) is MEF-exported as `IMainViewModel` and injected as the form's data context by `DefaultModuleController`.
- `UserManagement.ViewModel.IAddUserViewModel` / `IShowUsersViewModel` — the child view models the form binds to. `IAddUserViewModel` inherits `INotifyDataErrorInfo` (via `ValidatableViewModelBase`), which is what drives the validation-error display.
- `UserManagement.Model.UserModel` — the item type the `ListBox` formats for display.
- MEF (`System.ComponentModel.Composition`) — the `[Export]`/import machinery that connects this form to the controller and view models.

### Behavior & Details

**Layout (designer file).** Two `GroupBox` panels: `groupBox1` ("Current Users") holds `lstUsers`, a `ListBox`; `groupBox2` ("Add New User") holds `txtFirstName`, `txtLastName`, and a `btnSave` ("Save") button, plus `label1`/`label2`. An `ErrorProvider` with `ErrorBlinkStyle.NeverBlink` is attached to the form to show validation tooltips. `InitializeComponent()` is designer-generated and should not be edited by hand; `Dispose(bool)` disposes the `components` container, which the `ErrorProvider` lives in.

**View-model hookup.** The form has no logic of its own beyond wiring. Setting `DataContext` (which the module controller does after construction) casts the value to `IUserViewModel` and calls `SetupBindings()`. If the value is `null`, nothing is bound and the form stays inert.

**Bindings.** `SetupBindings` uses the legacy `Control.DataBindings` API:

- `txtFirstName`/`txtLastName` bind their `Text` to `addUserViewModel.FirstName`/`LastName` with `DataSourceUpdateMode.OnPropertyChanged`, so the view model sees each keystroke immediately (needed because `AddUserViewModel` re-validates per property and `SaveUserCommand`'s `CanExecute` depends on field state).
- `lstUsers` gets a `BindingSource` over `showUsersViewModel.Users` — not the `ObservableCollection` directly — because WinForms `ListBox` does not track `CollectionChanged`. A handler on `Users.CollectionChanged` calls `bindingSource.ResetBindings(false)`, marshalled through `Invoke` when it fires off the UI thread (as it does when `ShowUsersViewModel` receives a `UserChangedMessage` from the messenger).
- A `Format` event renders each item as `"FirstName LastName"`, trimmed.

**Validation display.** The form subscribes to `addUserViewModel.ErrorsChanged`. For each changed property it resolves the matching control (`txtFirstName` or `txtLastName`), fetches the current error strings via `GetErrors(propertyName)`, and calls `errorProvider.SetError` with the first message — or an empty string, which clears the icon. Only those two properties are mappable; any other erroring property is silently ignored.

**Commands.** `btnSave.Click` checks `SaveUserCommand.CanExecute(null)` and then calls `Execute(null)`. `SaveUserCommand` is an `IAsyncRelayCommand` generated from `AddUserViewModel`'s `[RelayCommand]` method, so `Execute` runs the async save and returns immediately; there is no busy state or disable-while-saving in this form (the WPF view does disable its button on `IsRunning`). On form `Load`, the form calls `showUsersViewModel.LoadedCommand.ExecuteAsync(null)` to trigger the initial user fetch.

### Rough edges

- The form's `Text` is still the designer default `"Form1"`.
- The `IMainView<TDialogResult, TDimension>` members the form must expose (`Left`, `Top`, `Width`, `Height`) are not implemented in either partial — the code is expected to compile against `Form`'s matching members (e.g. `Form.Height` is `int` while `Form.Top` is `int?`), but no implicit conversions were verified, so if this compiles it does so by member name rather than by an obvious match.
- `SetupBindings` is called every time `DataContext` is set, with no cleanup of prior event subscriptions; assigning a second view model would stack duplicate handlers on the controls. In practice `DefaultModuleController` sets it once.
- `SaveUserCommand` is invoked via synchronous `Execute` with the task result unobserved; a failure inside `SaveUserAsync` (the view model rethrows `BusinessLayerException`) surfaces as an unhandled exception rather than as form feedback.