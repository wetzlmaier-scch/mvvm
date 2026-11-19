### Purpose

`UserManagement/Design/DesignUserViewModel.cs` defines `DesignUserViewModel`, a design-time stand-in for the real `UserViewModel`. The WPF XAML designer cannot run MEF, so the `ShowUsersViewModel` and `AddUserViewModel` properties on the real `UserViewModel` (populated via `[Import]`) would be empty when `UserView.xaml` renders in the designer. This class exists so the window previews with live-looking sub-view models and sample data instead of a blank layout. It is never involved in runtime composition.

### Role in the Codebase

One of three designer-only stubs in `UserManagement/Design/` (with `DesignAddUserViewModel` and `DesignShowUsersViewModel`). Each mirrors a real view model from `UserManagement/ViewModel/` and is referenced only inside `d:` (design-time) XAML metadata. It is a leaf of the designer hierarchy — the root design-time object from which the two child stubs are constructed.

Removing it would not change application behavior at all; the designer would simply show the empty `UserView` layout without user list or form data, since the MEF imports would not be satisfied.

### Relationships

- **Base class:** `UserViewModel` (`UserManagement/ViewModel/UserViewModel.cs`), a `partial` class extending `MainViewModelBase` and implementing `IUserViewModel`, exported via MEF as both `IUserViewModel` and `IMainViewModel`. `DesignUserViewModel` inherits its two import properties: `ShowUsersViewModel` (`IShowUsersViewModel`) and `AddUserViewModel` (`IAddUserViewModel`).
- **Child stubs (same directory):** the constructor wires these in manually, bypassing MEF entirely:
  - `ShowUsersViewModel = new DesignShowUsersViewModel()` — seeds the `Users` collection with three sample `UserModel` entries.
  - `AddUserViewModel = new DesignAddUserViewModel()` — pre-fills `FirstName`/`LastName` with "Sample User".
- **Consumer:** `WpfApp1/View/UserView.xaml` is the only reference in the repo, via `d:DataContext="{d:DesignInstance Type=design:DesignUserViewModel, IsDesignTimeCreatable=True}"`. `IsDesignTimeCreatable=True` means the designer builds this class through its parameterless constructor. Because the attribute is in the ignorable `d` namespace (`mc:Ignorable="d"`), the runtime build ignores it and composes the real `UserViewModel` through MEF instead.

### Behavior & Details

- The class is just a constructor with two assignments — no other members. Nothing is overridden from `UserViewModel`.
- Constructor ordering note: base `UserViewModel` declares both properties with `= null!` initializers and MEF `[Import]` attributes. The derived constructor overwrites those initial values immediately, so MEF is never asked to resolve the imports in this path.
- No commands, data, or validation come from this class itself; everything the designer binds against (`ShowUsersViewModel.Users`, `AddUserViewModel.FirstName`, `SaveUserCommand`, etc.) comes from the parent classes. So if you change the shape of `AddUserViewModel` or `ShowUsersViewModel` — a renamed property, a moved collection — the designer preview of `UserView` follows automatically, but anything this stub constructs that no longer compiles will break the design build even though the runtime app is unaffected.

### Rough edges

- `DesignUserViewModel` hard-wires its children via `new` rather than mirroring the runtime's MEF exports. That means the design-time tree is a second, hand-maintained composition that will drift silently if the runtime composition changes (e.g. a child type is swapped or renamed).
- `DesignAddUserViewModel` is declared `partial` with no other partial in the repository, while this class and `DesignShowUsersViewModel` are not — an inconsistent marker that currently has no effect.