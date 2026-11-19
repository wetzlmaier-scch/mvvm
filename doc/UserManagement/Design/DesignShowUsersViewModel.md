### Purpose

`UserManagement/Design/DesignShowUsersViewModel.cs` — `DesignShowUsersViewModel` is a design-time-only subclass of `ShowUsersViewModel` whose sole job is to give the WPF XAML designer something to render. At design time the real view model can't work: its `Users` collection is only filled by an async command (`OnLoadedAsync`) after its `IUserService` is composed in via MEF, neither of which exists in the designer. This subclass seeds `Users` with three hard-coded `UserModel` entries in its constructor so the designer preview shows a populated list.

### Role in the Codebase

The file lives in the `UserManagement/Design/` folder, which holds designer stand-ins for the real view models (alongside `DesignUserViewModel` and `DesignAddUserViewModel`). It is excluded from any runtime path — nothing in the composed application constructs it. If it were removed, nothing in the running application would break; the designer preview of the users list in `WpfApp1/View/UserView.xaml` would simply be empty.

### Relationships

- **Base class: `ShowUsersViewModel`** (`UserManagement/ViewModel/ShowUsersViewModel.cs`). It provides the inherited `Users` `ObservableCollection<UserModel>` that this class populates. Inheriting it (rather than duck-typing) is what makes the instance valid as the `ShowUsersViewModel` property of `DesignUserViewModel`.
- **`UserModel`** (`UserManagement/Model/UserModel.cs`) — the `ObservableObject` with `FirstName`/`LastName` that the demo entries are built from.
- **Consumer: `DesignUserViewModel`** (same directory) — the only in-repo user. Its constructor does `ShowUsersViewModel = new DesignShowUsersViewModel();`. `DesignUserViewModel` is itself referenced from `WpfApp1/View/UserView.xaml` via `d:DataContext="{d:DesignInstance Type=design:DesignUserViewModel, IsDesignTimeCreatable=True}"`, which is why `IsDesignTimeCreatable=True` works: this class has a public parameterless constructor.

### Behavior & Details

- The constructor adds three `UserModel` instances — Jane Doe, John Smith, Alice Johnson — to the inherited `Users` collection. The names are placeholders with no runtime meaning.
- Nothing else is overridden. By construction, all of the base class's behavior is implicitly inherited, which includes two things worth knowing:
  - The base constructor registers a `UserChangedMessage` handler on `WeakReferenceMessenger`. So even the design-time instance subscribes to that messenger (harmless in the designer, but a live subscription it never unregisters, since the base class has no disposal logic).
  - The `[Import]` properties `UserService` and `LoggingService` remain `null!` — no MEF container composes a design instance. Calling `OnLoadedAsync` on a design instance would throw; the designer never does.

### Rough edges

- The demo entries are created as plain objects with no `Id` or other identifying data beyond names, so any designer control bound beyond name properties would render blank — fine for the current views, but a constraint if the designer preview is ever extended to show more fields.