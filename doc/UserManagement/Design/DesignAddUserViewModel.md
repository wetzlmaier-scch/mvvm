### Purpose

`UserManagement/Design/DesignAddUserViewModel.cs`

`DesignAddUserViewModel` exists so the "add user" form renders with plausible sample content in a designer or preview tool. It subclasses the real `AddUserViewModel` and overrides only the constructor to pre-fill `FirstName` and `LastName` with `"Sample"` and `"User"`. This is a WPF design-time stub, not runtime application code.

### Role in the Codebase

It lives in the `UserManagement/Design/` folder alongside `DesignShowUsersViewModel` and `DesignUserViewModel`, a sibling design-time stub for each of the real view models in `UserManagement/ViewModel/`. Its parent `AddUserViewModel` (`UserManagement/ViewModel/AddUserViewModel.cs`) is a partial class; the XAML designer generates a second partial for design purposes, and this subclass is what the designer instantiates in place of it.

The sole consumer is the XAML designer itself. `WpfApp1/View/UserView.xaml` designates `DesignUserViewModel` via `d:DesignInstance`, and `DesignUserViewModel`'s constructor constructs a `DesignAddUserViewModel` for its `AddUserViewModel` property. At runtime nothing instantiates this class.

### Relationships

- Inherits `AddUserViewModel` from `UserManagement/ViewModel/AddUserViewModel.cs`, which in turn extends `ValidatableViewModelBase<IUser>` and exports `IAddUserViewModel`. `DesignAddUserViewModel` adds no members beyond the constructor, so it inherits all behavior: the `FirstName`/`LastName` observable properties, validation, and `SaveUserAsync`.
- Used only by `DesignUserViewModel` (same folder), which sets it as the `AddUserViewModel` property when the designer builds the view hierarchy.

### Behavior & Details

The constructor sets the two inherited properties to non-empty values so the text fields are visibly populated and the inherited `SaveUserCommand` appears executable in the designer. Because these assignments go through the inherited property setters, `OnFirstNameChanged`/`OnLastNameChanged` run and call `ValidateProperty`; the constructor does not touch the injected `UserService`, `LoggingService`, or `MessageService`, which remain null (`null!`) and are only ever used inside `SaveUserAsync` — which the designer does not invoke.

One caveat: `IsDesignTimeCreatable=True` in `UserView.xaml` only guarantees constructability, not safety. If any designer action ever executes `SaveUserAsync`, the null-injected services would throw — but a plain designer preview only builds the object graph and binds properties, so this never happens in practice.