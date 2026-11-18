`UserManagement/Design/DesignUserViewModel.cs` defines `DesignUserViewModel`, a design-time stand-in for the real `UserViewModel`. The WPF XAML designer cannot run MEF, so this class exists so that `UserView.xaml` can show a fully populated screen in the designer.

### Purpose
`UserView.xaml` sets its design data context to `DesignUserViewModel` via `d:DesignInstance`:

```xml
d:DataContext="{d:DesignInstance Type=design:DesignUserViewModel, IsDesignTimeCreatable=True}"
```

`IsDesignTimeCreatable=True` means the designer instantiates this class through its parameterless constructor at design time. At runtime the same XAML file is composed with the real `UserViewModel` instead; nothing in the application wires up `DesignUserViewModel`.

The class exists to bridge a gap: the real view model's child properties are filled by MEF `[Import]`, which never runs in the designer. This class fills them manually so the designer can render the view.

### Role in the Codebase
It is the root of the three design-time view models in the `UserManagement/Design` folder. Its constructor composes the two children:

- `ShowUsersViewModel = new DesignShowUsersViewModel()` — populates `Users` with three sample `UserModel` entries (Jane Doe, John Smith, Alice Johnson).
- `AddUserViewModel = new DesignAddUserViewModel()` — sets `FirstName = "Sample"`, `LastName = "User"`.

Both design-time children are plain subclasses of the real `AddUserViewModel` / `ShowUsersViewModel`, so all XAML bindings (`AddUserViewModel.FirstName`, `ShowUsersViewModel.Users`, `AddUserViewModel.SaveUserCommand`, etc.) resolve against the design instance exactly as they do at runtime. Removing this class would leave the designer with an empty `UserView.xaml` — no runtime behaviour depends on it, and no C# code references it.

### Relationships
- **Base type:** `UserViewModel` (`UserManagement/ViewModel/UserViewModel.cs`), which derives from `MainViewModelBase` (`Mvvm/MainViewModelBase.cs`) and implements `IUserViewModel`. `UserViewModel` declares `AddUserViewModel` and `ShowUsersViewModel` as public `[Import]` properties; this class does not override anything, it only assigns them in its constructor.
- **Design-time children:** `DesignAddUserViewModel` and `DesignShowUsersViewModel` (same folder), which subclass the real `AddUserViewModel` and `ShowUsersViewModel` and seed demo values.
- **Consumer:** `WpfApp1/View/UserView.xaml` is the only thing that names `DesignUserViewModel`, and only inside `d:DataContext` (design-time metadata, `mc:Ignorable="d"`), so it is invisible to the compiled application.

Note the inheritance side effects: `MainViewModelBase`'s constructor bails out immediately when `IsInDesignMode` is true (no window hook-up, no icon loading), and its `Shutdown` / `CloseWindow` methods are no-ops in design mode, so constructing this class in the designer is safe and has no application side effects.

### Behavior & Details
- The only logic is the constructor: it assigns both inherited child properties. The inherited `[Import]` attributes are not evaluated here — MEF never touches this class, which is exactly the point.
- One difference from the real object: `DesignShowUsersViewModel`'s constructor registers a `UserChangedMessage` handler on `WeakReferenceMessenger` (inherited from `ShowUsersViewModel`'s constructor). That registration is harmless in the designer — no messages are ever sent there — but the sample users in the designer view are static; they do not react to anything.
- The demo values are display-only and carry no meaning at runtime; `DesignAddUserViewModel`'s `Sample`/`User` names and the three names in `DesignShowUsersViewModel` are just placeholders for the designer canvas.