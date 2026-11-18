### Purpose

`Dto/IUser.cs` defines `IUser`, a two-property contract for anything that carries a user's first and last name. It exists so the presentation and validation layers can talk about "a user" in terms of just the name fields, without depending on the concrete `UserDto` with its `Id` or on the persistence entities.

### Role in the Codebase

It is a small shared contract at the bottom of the layering: the `Dto` project defines the interface and `UserDto` (in `Dto/UserDto.cs`) is its only concrete implementation. The view models and validation live in terms of `IUser` rather than `UserDto`, which lets a form or validator bind to name-only data — for example, a new user being typed in that doesn't have an `Id` yet — before it is packaged into a `UserDto` for the service layer.

### Relationships

- `Dto/UserDto.cs` — `UserDto` implements `IUser` and adds `Id`; it is the object `IUserService` (in `BusinessLayer/IUserService.cs`) actually accepts and returns.
- `Dto/Validation/UserValidator.cs` — a FluentValidation `AbstractValidator<IUser>` that requires `FirstName` and `LastName` to be non-empty.
- `UserManagement/ViewModel/IAddUserViewModel.cs` — extends `IUser`, so `AddUserViewModel` inherits the `FirstName`/`LastName` properties and derives from `ValidatableViewModelBase<IUser>(new UserValidator())`, which applies the validator to the view model itself.
- `IUserService` uses `UserDto` rather than `IUser` at its boundary; `IUser` is the view-model-side shape.

### Behavior & Details

The interface has no logic: just `string FirstName { get; set; }` and `string LastName { get; set; }`. Both are read/write, which is intentional — `AddUserViewModel`'s `[ObservableProperty]`-generated properties back the interface with change notification, and the base view model's validation is run against the implementing object. `UserDto` marks the same properties `required`, so a `UserDto` cannot be constructed without names, while the interface itself imposes no such constraint.

Note the property set differs from `UserDto` on purpose: `IUser` deliberately omits `Id`, since the add-user flow has no identity to expose at the form level.