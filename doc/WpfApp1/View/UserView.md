# WpfApp1/View/UserView.xaml

The `UserView` window is the main view of the WPF host application (`WpfApp1`). The group consists of two files compiled into one partial class:

- `WpfApp1/View/UserView.xaml` — the markup: a two-column window with a user list on the left and an "Add New User" form on the right, all driven by data bindings.
- `WpfApp1/View/UserView.xaml.cs` — the code-behind: an empty constructor calling `InitializeComponent()`, plus the class declaration and MEF export attribute.

## Purpose

`UserView` is the window through which a user of the WPF app browses active users and adds a new one. It contains no logic of its own; it is a pure presentation surface whose two halves are bound to the `UserManagement` module's view models. It exists so that the `UserManagement` feature has a WPF face — the twin of `UserForm` in `WinFormsApp1`, which implements the same role for the WinForms host.

## Role in the Codebase

`UserView` is the main window of `WpfApp1`. It is not created by any code in this repository directly; instead it advertises itself to MEF via `[Export(typeof(IMainView<bool?, double>))]`. The WPF host's `DefaultModuleController` (`WpfApp1/Controller/DefaultModuleController.cs`) imports exactly that contract, sets the composed `UserViewModel` as the window's `DataContext` in its constructor, then shows it via `MainView.Show()` in `Run()` and closes it via `MainView.Close()` in `Shutdown()`. Remove or rename the export and the WPF host has no main window to run at all.

The interface chain it implements: `IMainView<bool?, double>` (`Mvvm/IMainView.cs`) extends `IDialogView<bool?>` (`Mvvm/IDialogView.cs`), which extends `IView` (`Mvvm/IView.cs`). All members come from WPF's `Window` itself — `DataContext`, `DialogResult`, `Closing`/`Closed`, `Show()`, `Close()`, and the `double`-typed `Left`/`Top`/`Width`/`Height` — so `UserView` implements the interface implicitly with no explicit code. The `<bool?, double>` type arguments mirror WPF's `bool?` `Window.DialogResult` and `double` geometry units, in contrast to `UserForm`'s `<DialogResult, int>`.

## Relationships

- **`UserViewModel`** (`UserManagement/ViewModel/UserViewModel.cs`) — the runtime `DataContext`. The controller assigns it; the XAML binds to its `AddUserViewModel` and `ShowUsersViewModel` properties and to its inherited `Icon` property (`Window.Icon="{Binding Icon}"`).
- **`AddUserViewModel`** (`UserManagement/ViewModel/AddUserViewModel.cs`) — backs the form: `FirstName`, `LastName` (both validated via `INotifyDataErrorInfo`, which `IAddUserViewModel` extends) and `SaveUserCommand` (a generated `RelayCommand`, so the button can observe `IsRunning`).
- **`ShowUsersViewModel`** (`UserManagement/ViewModel/ShowUsersViewModel.cs`) — backs the list: its `Users` collection and its `LoadedCommand` (generated from `OnLoadedAsync` by the CommunityToolkit `[RelayCommand]`).
- **`DesignUserViewModel`** (`UserManagement/Design/DesignUserViewModel.cs`) — design-time-only. Referenced solely in `d:DataContext` so the XAML designer can render the window without a live composition container; it has no runtime effect.
- **`DebugConverter`** (`Common/Converter/DebugConverter.cs`) — declared as the `MyDebugConverter` resource, but its only consuming binding is commented out (see Rough edges).
- **`b:Interaction` / `b:EventTrigger` / `b:InvokeCommandAction`** — the WPF Behavior extension kit, used to hook the window's `Loaded` event to a command without code-behind.

## Behavior & Details

**Layout.** A 700×450, center-screen window on a light-gray background. A two-column `Grid` (widths `2*` and `1.5*`) places a white rounded `Border` holding the "Current Users" `ListView` on the left, and a second `Border` with the "Add New User" form on the right. A window-level `Style` gives every `TextBox` uniform padding and border.

**Loading the user list.** On the window's `Loaded` event, a behavior trigger invokes `ShowUsersViewModel.LoadedCommand`, which asynchronously fetches active users through `IUserService.GetActiveUsers()` and populates the `Users` collection the `ListView` binds to. The `ListView` shows `FirstName` and `LastName` columns of `UserModel` items. This is the one piece of control flow the view contributes: without this trigger, the list would stay empty, because `ShowUsersViewModel` only loads on demand.

**Adding a user.** Both `TextBox`es bind with `UpdateSourceTrigger=PropertyChanged`, so `AddUserViewModel.FirstName`/`LastName` change (and revalidate) as the user types; each change also flips `SaveUserCommand.CanExecute` via the generated `[NotifyCanExecuteChangedFor]` wiring. The Save button runs `AddUserViewModel.SaveUserCommand`, which validates, calls `UserService.AddUser`, broadcasts a `UserChangedMessage` (which `ShowUsersViewModel` listens for to append the new row), and clears the fields. A `DataTrigger` on `SaveUserCommand.IsRunning` swaps the button label to "Saving..." while the async command runs, and a `Trigger` dims it while disabled.

**Validation display — asymmetric by accident.** Only the first-name box has an error template: it uses `ValidatesOnNotifyDataErrors=True` plus `Validation.ErrorTemplate="{StaticResource InlineErrorTemplate}"`, which shows the first error's text in red below the box. The last-name box uses the same binding options but displays errors only as a `ToolTip` on the box itself. There is no indication in the markup that this difference is intentional.

## Rough edges

- The `MyDebugConverter` resource and the commented-out `TextBox` that used it are dead markup. Note that `DebugConverter.Convert` calls `Debugger.Break()`, so uncommenting the binding would stop the app in the debugger on every keystroke.
- Error feedback differs between the two fields (inline error vs. tooltip), and nothing in the code suggests the difference is deliberate.