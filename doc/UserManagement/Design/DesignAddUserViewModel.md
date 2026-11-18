# `UserManagement/Design/DesignAddUserViewModel.cs`

### Purpose

`DesignAddUserViewModel` exists so the "add user" form has plausible content to render in the WPF XAML designer. It subclasses the runtime `AddUserViewModel` and pre-fills `FirstName` with `"Sample"` and `LastName` with `"User"`, so the form's text boxes appear populated rather than empty when the designer loads the view. It contains no other logic.

### Role in the Codebase

One of three designer-only view models in `UserManagement/Design/` (alongside `DesignUserViewModel` and `DesignShowUsersViewModel`). The class is marked `partial` but no generated designer partial exists for it, so the constructor shown here is its complete implementation. Nothing at runtime references it: the runtime container exports `AddUserViewModel` via `[Export(typeof(IAddUserViewModel))]`, and the designer path never goes through the container. If it were removed, the add-user region of the designer preview would show empty fields; runtime behaviour would be unaffected.

### Relationships

- **Base class:** `AddUserViewModel` (`UserManagement/ViewModel/AddUserViewModel.cs`), itself a `ValidatableViewModelBase<IUser>` with `[ObservableProperty]`-generated `FirstName`/`LastName` properties, an `IUserService` import, and a `SaveUserCommand`. `DesignAddUserViewModel` overrides nothing — it only assigns to the inherited properties in its constructor.
- **Sole constructor:** `DesignUserViewModel` (`UserManagement/Design/DesignUserViewModel.cs`) sets `AddUserViewModel = new DesignAddUserViewModel()`.
- **Design-time chain:** `WpfApp1/View/UserView.xaml` declares `d:DataContext="{d:DesignInstance Type=design:DesignUserViewModel, ...}"`, so the designer constructs `DesignUserViewModel`, which in turn constructs this class.

### Behavior & Details

The single constructor assigns two string literals to inherited observable properties. Because those properties are `[ObservableProperty]`-generated, the assignments also trigger the base class's `OnFirstNameChanged`/`OnLastNameChanged` handlers, which call `ValidateProperty`. That validation runs against a `UserValidator` and `HasErrors`, so it is worth knowing the sample values ("Sample"/"User") pass validation — otherwise the designer would display validation error adornments. No commands, services, or logging are touched here; the `[Import]` services on the base class remain null in this design-time instance, which is fine because nothing calls them from the designer.