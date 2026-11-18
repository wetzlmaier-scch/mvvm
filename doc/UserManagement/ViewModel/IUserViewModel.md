# UserManagement/ViewModel/IUserViewModel.cs

### Purpose

`IUserViewModel` is the top-level view-model contract for the UserManagement feature. It exists so that host code — the MEF composition root and the WinForms view — can depend on a stable interface for the feature's view-model tree instead of the concrete `UserViewModel` class. Because the module is wired through MEF, the interface is also the lookup key: `UserViewModel` registers itself with `[Export(typeof(IUserViewModel))]`, and consumers resolve it back by this same type.

The interface holds two read-only properties, each exposing one of the feature's two sub-features:

- `AddUserViewModel` (`IAddUserViewModel`) — the child view model for creating a user (name fields, `SaveUserCommand`, `SaveUserAsync()`).
- `ShowUsersViewModel` (`IShowUsersViewModel`) — the child view model for listing the active users (`Users` collection, `LoadedCommand`, `OnLoadedAsync()`).

It is a façade: the interface itself carries no state or behaviour, only the composition root of the feature's view-model hierarchy.

### Role in the Codebase

Inside `UserManagement/ViewModel`, the folder follows an interface/class pair pattern per view model (`IAddUserViewModel`/`AddUserViewModel`, `IShowUsersViewModel`/`ShowUsersViewModel`, and this interface with `UserViewModel`). This file is the "shell" contract of that group: the only interface that names both children, so it is what the rest of the application reaches for first.

If it were removed, `UserViewModel` would no longer be resolvable as the feature entry point: `ConsoleApp1/Program.cs` calls `container.GetExportedValues<IUserViewModel>().SingleOrDefault()` at startup, and `WinFormsApp1/View/UserForm.cs` casts its `DataContext` to `IUserViewModel?` in `SetupBindings`. Both hosts and the export attribute on `UserViewModel` reference this type directly.

### Relationships

- **Implemented by** `UserViewModel` (`UserManagement/ViewModel/UserViewModel.cs`), the only implementation. That class extends `MainViewModelBase` and satisfies the two properties via `[Import]`ed properties; it also carries a second export, `[Export(typeof(IMainViewModel))]`, so the same instance can be resolved as a generic main view model (used by `WpfApp1`'s `DefaultModuleController`).
- **Depends on** `IAddUserViewModel` and `IShowUsersViewModel`, both defined in the same folder. The interface has no other dependencies; the `using System.ComponentModel.Composition;` directive in this file is unused — no MEF attributes appear on the interface itself.
- **Consumed by**:
  - `ConsoleApp1/Program.cs` — resolves the export and drives the feature through `userViewModel.AddUserViewModel` (sets `FirstName`/`LastName`, calls `SaveUserAsync()` in a loop) and `userViewModel.ShowUsersViewModel` (`OnLoadedAsync()`, then iterates `Users`).
  - `WinFormsApp1/View/UserForm.cs` — receives it as `DataContext` and binds its controls through the two child view models.

### Behavior & Details

There is no behaviour to trace — the interface has exactly two get-only properties and no methods, events, or inherited members. The notable contract points:

- The properties are read-only here, but in `UserViewModel` they are `[Import]`ed with public setters (`{ get; set; } = null!`). MEF satisfies them at composition time; if no matching export exists, MEF fails to compose the container rather than the property staying null.
- Both property names match the property names on `UserViewModel` exactly, which is what allows MEF to import against the concrete class while the interface stays free of any MEF or base-type coupling.

### Rough edges

The `using System.ComponentModel.Composition;` at the top of this file is never used by any member in the interface — a leftover, presumably copied from the implementation file.