# UserManagement/ViewModel/IUserViewModel.cs

### Purpose

`IUserViewModel` is the top-level view-model contract for the user-management feature. It exists so that composition-root code and host views can depend on a stable interface rather than the concrete `UserViewModel` class. The interface exposes the two child view models that make up the feature — `AddUserViewModel` and `ShowUsersViewModel` — as get-only properties, so a consumer can reach the add-user form logic and the user-list logic through a single object.

### Role in the Codebase

It is the umbrella contract inside the `UserManagement/ViewModel` folder, which also contains the interfaces `IAddUserViewModel` and `IShowUsersViewModel` plus their implementations. `UserViewModel` in the same folder is the only class implementing `IUserViewModel`.

Because the app is wired via MEF, the interface is more than an abstraction: it is the lookup key. `UserViewModel` carries `[Export(typeof(IUserViewModel))]` (and also `[Export(typeof(IMainViewModel))]`), and the console host retrieves the view model purely by this type. Without the interface, the composition container would have nothing to ask for.

### Relationships

- **Implementor:** `UserViewModel` (same folder) implements it, exposing the two child view models as MEF `[Import]` properties: `IAddUserViewModel AddUserViewModel` and `IShowUsersViewModel ShowUsersViewModel`, both typed `null!` and populated by the container at composition time. `UserViewModel` also derives from `MainViewModelBase` (in the `Mvvm` project) and implements `IMainViewModel`, but those members are not part of this interface.
- **Consumers:**
  - `ConsoleApp1/Program.cs` calls `container.GetExportedValues<IUserViewModel>()` and throws if nothing is found. It then drives the feature through the interface: it sets `FirstName`/`LastName` on `userViewModel.AddUserViewModel` and awaits `SaveUserAsync()` in a loop, then awaits `ShowUsersViewModel.OnLoadedAsync()` and iterates `ShowUsersViewModel.Users`.
  - `WinFormsApp1/View/UserForm.cs` casts its `DataContext` to `IUserViewModel?` in `SetupBindings` and binds text boxes, the list box, and the error provider through the two child view models obtained from the interface.
- **Child interfaces:** `IAddUserViewModel` (which itself extends `INotifyDataErrorInfo` and `IUser`) and `IShowUsersViewModel` are defined in the same folder; this interface only exposes them, their members are not repeated here.

### Behavior & Details

The interface has no logic — it declares exactly two read-only properties:

- `IAddUserViewModel AddUserViewModel { get; }` — the view model for creating a user (fields, validation, `SaveUserCommand`/`SaveUserAsync`).
- `IShowUsersViewModel ShowUsersViewModel { get; }` — the view model for listing users (`Users` collection, `OnLoadedAsync`).

There are no methods, events, or default implementations. Note the properties are get-only, so consumers cannot swap in alternative child view models — the composition of the two children is fixed by `UserViewModel`'s imports.

The file imports `System.ComponentModel.Composition` but uses no MEF attribute; the import is unused in this file (the `[Export]`/`[Import]` attributes live in `UserViewModel.cs`).

### Rough edges

`IUserViewModel` is declared in the `UserManagement.ViewModel` namespace but is essentially a façade over two unrelated child view models; a consumer who needs, say, just the user list must go through the umbrella object. The unused `System.ComponentModel.Composition` import also suggests it was copy-carried from `UserViewModel.cs`.