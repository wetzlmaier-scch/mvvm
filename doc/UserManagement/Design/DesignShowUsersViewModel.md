# `UserManagement/Design/DesignShowUsersViewModel.cs`

### Purpose

`DesignShowUsersViewModel` is a design-time-only subclass of `ShowUsersViewModel` that gives the WPF XAML designer something to render. The real view model loads its user list from the business layer at runtime, so the designer — which can't do that — would show an empty view. This class overrides nothing and simply seeds the inherited `Users` collection with three hard-coded `UserModel` entries (Jane Doe, John Smith, Alice Johnson) in its constructor.

### Role in the Codebase

It is one of three design-time stubs in `UserManagement/Design/` (alongside `DesignUserViewModel` and `DesignAddUserViewModel`), each mirroring a real view model from `UserManagement/ViewModel/` with fake data. The class is never instantiated at runtime: it is only referenced from `DesignUserViewModel`'s constructor, which assigns it to the `ShowUsersViewModel` property in place of the MEF import that the real `UserViewModel` uses. Removing it would only degrade the designer preview; runtime behaviour is untouched.

### Relationships

- **`ShowUsersViewModel`** (`UserManagement/ViewModel/ShowUsersViewModel.cs`) — the base class. It provides the `Users` `ObservableCollection<UserModel>` this class populates, plus `[Import]` properties (`UserService`, `LoggingService`) and a `UserChangedMessage` messenger subscription in its own constructor. Note that `DesignShowUsersViewModel`'s constructor does **not** call `base()`, and `ShowUsersViewModel`'s parameterless constructor does the messenger registration — but since this class is only built by the designer, that registration is never exercised.
- **`UserModel`** (`UserManagement/Model/UserModel.cs`) — the data item type; a simple observable `FirstName`/`LastName` pair.
- **`DesignUserViewModel`** (same directory) — the only in-repo consumer; it constructs `new DesignShowUsersViewModel()` in its constructor, which is itself used via `d:DesignInstance` in the XAML.

### Behavior & Details

The entire class is a parameterless constructor that adds three `UserModel` instances to the inherited `Users` collection. There is no other logic, no overridden members, and no side effects beyond populating the collection.

### Rough edges

`ShowUsersViewModel` is declared `partial` but has no other part, and its constructor registers a `WeakReferenceMessenger` handler that throws `NotImplementedException` for `Updated`/`Deleted` user changes — harmless here because this design-time subclass is only ever constructed by the designer, but worth knowing if someone ever instantiates it at runtime.