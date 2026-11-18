### Purpose

`WinFormsApp1/View/UserForm.Designer.cs` is the auto-generated half of the `UserForm` Windows Forms class. It contains no business logic of its own — it is the layout half of a split-class pattern the Visual Studio designer uses: this file instantiates and positions the form's controls, while the hand-written `UserForm.cs` wires them up to the view model.

### Role in the codebase

This file is a `partial class UserForm` declaration that pairs with `WinFormsApp1/View/UserForm.cs`. The two files together form one `Form`-derived class that implements `IMainView<DialogResult, int>` (a WinForms/MVVM shell view, exported via MEF with `[Export(typeof(IMainView<DialogResult, int>))]`). The View folder holds only these two files.

The form's business purpose, per the paired code-behind, is user management: a list box showing current users on the left, and an "Add New User" panel with first/last name fields and a Save button on the right.

`InitializeComponent()` is called from the constructor in `UserForm.cs`, so the form cannot display any controls without this file. The form text is set to `"Form1"` — the designer's default, never updated.

### Relationships

- **`UserForm.cs`** (same directory): the other half of the `partial class`. It declares the class's base (`Form`), its `IMainView<DialogResult, int>` implementation, and `SetupBindings(...)`, which is where every control declared here gets behaviour:
  - `txtFirstName` / `txtLastName` — `DataBindings` to `AddUserViewModel.FirstName` / `.LastName` with `DataSourceUpdateMode.OnPropertyChanged`.
  - `lstUsers` — bound through a `BindingSource` to `ShowUsersViewModel.Users`, with a `Format` handler rendering `"First Last"`.
  - `errorProvider` — fed from `AddUserViewModel.ErrorsChanged` to show validation messages next to the name fields.
  - `btnSave` — its `Click` executes `AddUserViewModel.SaveUserCommand`.
- Because `UserForm` is MEF-exported as `IMainView<DialogResult, int>`, the form is instantiated by the application's composition container, not by direct `new` calls from other code.

### Behavior & Details

- **`InitializeComponent()`** is the only method. It creates and positions nine controls. It must not be edited by hand — the designer regenerates it. The file also contains a boilerplate `Dispose(bool)` override that releases the `IContainer` if it is non-null.
- **Layout**, fixed pixel coordinates with `AutoScaleMode.Font`:
  - `groupBox1` ("Current Users", at 25,27, 329×345) contains `lstUsers`.
  - `groupBox2` ("Add New User", at 388,27, 346×345) contains `label1`/`txtFirstName` ("First Name"), `label2`/`txtLastName` ("Last Name"), and `btnSave` ("Save").
- **`errorProvider`** is created with the shared `components` container and set to `ErrorBlinkStyle.NeverBlink` — validation errors appear as a static icon/tooltip, not a blinking one.
- No control is given an event handler in this file; all behaviour is attached in `SetupBindings` in the code-behind.

Rough edges: the form's title remains the designer default `Text = "Form1"` rather than a descriptive caption.