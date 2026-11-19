### Purpose

`UserManagement/ViewModel/IAddUserViewModel.cs`

`IAddUserViewModel` is the contract for the "add a new user" screen of the User Management module. It defines what other parts of the application can rely on from that screen: the two user fields being entered, the command that saves them, the method that performs the save, and the data-error reporting the XAML binds against for field-level validation.

### Role in the Codebase

This is the interface half of the interface/class pair pattern the `UserManagement/ViewModel` folder follows for every view model (`IAddUserViewModel`/`AddUserViewModel`, `IShowUsersViewModel`/`ShowUsersViewModel`, `IUserViewModel`/`UserViewModel`). The concrete implementation, `AddUserViewModel` (`UserManagement/ViewModel/AddUserViewModel.cs`), is registered with MEF via `[Export(typeof(IAddUserViewModel))]`. The module composes its view models through MEF rather than direct instantiation, so the interface is the unit that MEF hands out and consumers bind to.

If it were removed, `AddUserViewModel` could no longer be exported or resolved, and the parent `UserViewModel` could not declare its child of this type.

### Relationships

This interface inherits from two parents:

- **`IUser`** (`Dto/IUser.cs`) — a small data shape from the `Dto` project with `FirstName` and `LastName`. Because of this, `IAddUserViewModel` (and therefore the view model) is itself a valid `IUser`, which is what `AddUserViewModel` also exploits when it extends `ValidatableViewModelBase<IUser>`.
- **`INotifyDataErrorInfo`** (`System.ComponentModel`, .NET framework type) — the WPF data-binding validation contract. The implementation's `ValidatableViewModelBase<IUser>` base class supplies it, and the XAML binds to it: `WpfApp1/View/UserView.xaml` sets `ValidatesOnNotifyDataErrors=True` on the first- and last-name text boxes.

Its members:

- `UserService` — a setter-injectable (MEF `[Import]` in the implementation) `IUserService` (`BusinessLayer/IUserService.cs`) that provides `AddUser(UserDto)` for the actual persistence.
- `LoggingService` — an `ILoggingService` (`Logging/ILoggingService.cs`) likewise import-injected into the implementation.
- `SaveUserCommand` — a CommunityToolkit `IAsyncRelayCommand` generated in the implementation by the `[RelayCommand]` source generator from `SaveUserAsync`. The view binds its `Command` and `IsRunning` (a disabled-while-saving state trigger) to it.
- `SaveUserAsync()` — the programmatic entry point for the save; the command wraps this method.

The consumer is the parent shell of the module: `IUserViewModel` declares `IAddUserViewModel AddUserViewModel { get; }`, and `UserViewModel` imports it via MEF. The view model is not referenced from XAML directly; the view binds to the parent and reaches it through `AddUserViewModel.*` (e.g. `{Binding AddUserViewModel.FirstName}`, `{Binding AddUserViewModel.SaveUserCommand}` in `UserView.xaml`).

### Behavior & Details

The interface is a declaration only — all behaviour lives in `AddUserViewModel`, but knowing the split is important:

- `SaveUserAsync()` is implemented as the command body: it validates `FirstName`/`LastName`, shows an error via `IMessageService` if invalid, calls `UserService.AddUser(...)`, broadcasts a `UserChangedMessage` over `WeakReferenceMessenger`, then clears the fields. Consumers of the interface can trigger the save programmatically, but they cannot observe the `UserChangedMessage` broadcast from this contract alone.
- `UserService` and `LoggingService` are declared as get/set here even though the interface consumers never assign them; in the implementation they are one-way MEF `[Import]` properties. The interface only exists so consumers *can* reach them.
- Because the interface extends `IUser`, callers holding an `IAddUserViewModel` can read `FirstName`/`LastName` as if it were a plain `IUser`, without depending on the ViewModel type.

One rough edge: `LoggingService` is declared on the contract and imported by `AddUserViewModel`, but the class never actually uses it — the error path goes through `MessageService.ShowError` instead. A consumer reading the contract would reasonably expect logging to happen on save; it does not.