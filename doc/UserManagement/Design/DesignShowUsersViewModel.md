### Purpose

`UserManagement/Design/DesignShowUsersViewModel.cs` defines `DesignShowUsersViewModel`, a design-time-only subclass of `ShowUsersViewModel` whose sole job is to give the XAML editor fake data. When a designer (Visual Studio, Blend) opens `UserView.xaml`, the designer instantiates this class instead of the real ViewModel, and it seeds the user list with three hard-coded demo users so the "Current Users" `ListView` renders with sample rows rather than appearing empty.

### Role in the Codebase

This class is never used by the running application. The production code path composes `ShowUsersViewModel` via MEF (`[Export(typeof(IShowUsersViewModel))]` in `UserManagement/ViewModel/ShowUsersViewModel.cs`) and loads real data from `IUserService`. The design class exists purely to make the view previewable in the XAML Designer. Removing it would not affect the application at runtime — the XAML would simply show no users in the designer preview.

### Relationships

- **`ShowUsersViewModel`** (`UserManagement/ViewModel/ShowUsersViewModel.cs`) — the parent class, an MEF-exported MVVM ViewModel that exposes `Users`, an `ObservableCollection<UserModel>`, and a `LoadedCommand` that fetches active users from `IUserService`. The design subclass inherits all of this, including the constructor's `UserChangedMessage` messenger registration, and reuses the inherited `Users` collection.
- **`UserModel`** (`UserManagement/Model/UserModel.cs`) — a simple `ObservableObject` with `FirstName`/`LastName` properties; the demo rows are built from it.
- **`DesignUserViewModel`** (`UserManagement/Design/DesignUserViewModel.cs`) — the sibling design-time class that composes this one: its constructor assigns `ShowUsersViewModel = new DesignShowUsersViewModel()`, alongside `DesignAddUserViewModel`.
- **`UserView.xaml`** (`WpfApp1/View/UserView.xaml`) — the consumer. Its `d:DataContext` uses `d:DesignInstance Type=design:DesignUserViewModel, IsDesignTimeCreatable=True`, so the designer instantiates `DesignUserViewModel` (and therefore this class) only at design time. Because `IsDesignTimeCreatable` is set, the constructor runs in the designer.

### Behavior & Details

The entire logic is the parameterless constructor: it adds three fixed `UserModel` entries — Jane Doe, John Smith, and Alice Johnson — to the inherited `Users` collection. No other members are defined.

One behavioral note: the inherited `ShowUsersViewModel` constructor registers a `UserChangedMessage` handler on `WeakReferenceMessenger.Default`. Since the design-time instance runs the real constructor, that registration happens in the designer as well — harmless in practice (the designer will not raise those messages), but worth knowing if you ever reason about messenger subscribers. The inherited `LoadedCommand` is also present on the design instance; the XAML only invokes it via an `EventTrigger` on `Loaded`, which does not fire in the designer, so the real `IUserService` lookup (whose `[Import]` would be unpopulated) is never triggered at design time.