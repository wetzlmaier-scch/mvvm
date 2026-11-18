### Purpose

`UserManagement/Design/DesignUserViewModel.cs` defines `DesignUserViewModel`, a design-time stand-in for the real `UserViewModel`. The WPF XAML designer cannot run the application's MEF composition, so the designer's `[Import]`-wired child view models (`AddUserViewModel`, `ShowUsersViewModel`) would be `null` if it instantiated `UserViewModel` directly. This class exists so the XAML designer can create a `UserViewModel` that is actually populated with sample data, letting developers see the user view laid out with content.

### Role in the Codebase

It is one of three files in `UserManagement/Design/` that provide designer-only view models (alongside `DesignShowUsersViewModel` and `DesignAddUserViewModel`). The class is never used at runtime — the only reference to it is in `WpfApp1/View/UserView.xaml`:

```xml
d:DataContext="{d:DesignInstance Type=design:DesignUserViewModel, IsDesignTimeCreatable=True}"
```

The `d:` namespace means this `DesignInstance` markup applies only inside the designer. Removing this file (and its two siblings) would not change runtime behaviour at all; the designer would simply show the user view empty, with no child view models or demo data.

### Relationships

- **Base type:** `UserViewModel` (in `UserManagement/ViewModel/UserViewModel.cs`), which extends `MainViewModelBase` and implements `IUserViewModel`. It exposes the two child view model properties `AddUserViewModel` (`IAddUserViewModel`) and `ShowUsersViewModel` (`IShowUsersViewModel`), marked `[Import]` for MEF composition. `DesignUserViewModel` inherits these and does not override or add anything of its own beyond the constructor.
- **Constructs:** `DesignShowUsersViewModel` and `DesignAddUserViewModel`, both in the same `UserManagement/Design/` namespace. The former seeds its `Users` collection with three fake `UserModel` entries; the latter sets `FirstName`/`LastName` to "Sample User".
- **Consumed by:** the XAML designer via `UserView.xaml`; no production code path references it.

### Behavior & Details

The entire behaviour is the constructor, which assigns the two child view model properties that `UserViewModel` leaves as `null!` until MEF fills them:

```csharp
ShowUsersViewModel = new DesignShowUsersViewModel();
AddUserViewModel = new DesignAddUserViewModel();
```

Because the designer cannot run composition, these direct instantiations substitute for the `[Import]` wiring. No commands, bindings, or logic are redefined here — at design time the view shows the designer's demo data through these children.

Note that this class does not itself carry MEF exports; unlike `UserViewModel`, it is not `[Export]`-ed, consistent with its designer-only role.