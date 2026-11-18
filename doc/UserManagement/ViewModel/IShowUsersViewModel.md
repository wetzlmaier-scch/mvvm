# `UserManagement/ViewModel/IShowUsersViewModel.cs`

### Purpose

`IShowUsersViewModel` is the contract for the view model that displays the list of active users in the UserManagement module. It exists so the parent view model (`UserViewModel`) can hold a reference to the show-users view model by interface, letting MEF inject the concrete implementation at composition time without the parent depending on that class directly.

### Role in the Codebase

This is the interface half of the show-users view model pair in `UserManagement/ViewModel/`, where each feature has an `I*ViewModel.cs` contract alongside its implementation. `ShowUsersViewModel` (same folder) is the sole implementation and is tagged `[Export(typeof(IShowUsersViewModel))]`, so MEF resolves this interface to it. If the interface were removed, `ShowUsersViewModel` would have nothing to export as and `UserViewModel`'s `[Import]` could not be satisfied — the composition of the whole user management view would fail.

### Relationships

- **Implemented by** `ShowUsersViewModel` in `UserManagement/ViewModel/ShowUsersViewModel.cs`, which also inherits from `ViewModelBase` (in `Mvvm/ViewModelBase.cs`).
- **Consumed by** `UserViewModel` (same folder), which exposes an `[Import]` property `IShowUsersViewModel ShowUsersViewModel`; `IUserViewModel` declares the same property read-only.
- **Members reference:**
  - `UserModel` — the observable model in `UserManagement/Model/UserModel.cs` that the collection holds.
  - `IUserService` — defined in `BusinessLayer/IUserService.cs`; the implementation uses its `GetActiveUsers()` to populate the list.
  - `ILoggingService` — a logging contract from the `Logging` project (its declaration is not visible in this file's usage; the implementation exposes it as a settable property).
  - `IAsyncRelayCommand` — a CommunityToolkit.Mvvm type; the implementation's `[RelayCommand]` attribute on `OnLoadedAsync` generates `LoadedCommand` from that method.

### Behavior & Details

The interface has no behavior of its own — all members are contracts the implementation must satisfy, and the interesting part is what each member commits the implementor to:

- **`Users`** — read-only `ObservableCollection<UserModel>`. Because the interface only exposes a getter, consumers (WPF/WinForms views and tests) can bind to the collection but can only mutate it through the view model's own commands and its `UserChangedMessage` handling.
- **`UserService` / `LoggingService`** — get/set properties. In `ShowUsersViewModel` these are MEF `[Import]` properties initialized with `null!`, so they are set by the container after construction, not through the constructor.
- **`LoadedCommand`** — `IAsyncRelayCommand` backing the initial user fetch. Views trigger it when they open: `WpfApp1/View/UserView.xaml` via an `InvokeCommandAction` on load, `WinFormsApp1/View/UserForm.cs` on the form's `Load` event.
- **`OnLoadedAsync()`** — the method `LoadedCommand` wraps; in the implementation it awaits `UserService.GetActiveUsers()` and appends each result to `Users`. Declaring it on the interface means consumers (e.g. `TestProject1/UserViewModelTest.cs`) can call the async method directly without going through the command.

One naming nuance: the interface member is `LoadedCommand` while the implementation's method is `OnLoadedAsync` — the source generator derives the command name from the method name, and `WinFormsApp1/View/UserForm.cs` refers to it in a comment as "OnLoadedCommand". They are the same command; bind to `LoadedCommand`.

### Rough edges

`LoadedCommand`'s `CanExecute` is not declared anywhere in the interface or the implementation (the generated command reports `true`), yet `UserForm.cs` calls `CanExecute(null)` before invoking it — that check is effectively dead code as written.