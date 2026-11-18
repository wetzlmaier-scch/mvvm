### Purpose

`Dto/IUser.cs` defines `IUser`, a two-property contract for anything that carries a user's first and last name. It exists so the validation and view-model layers can work with "a user" as a name-bearing shape without needing the full persistence model. The interface has no methods — only `FirstName` and `LastName` string properties — so it captures just the data a form displays and a validator checks.

### Role in the Codebase

`IUser` is the shared contract at the bottom of the layering in the `Dto` project. It is the minimal "user shape" that the UI side of the application programs against, while the business layer (`BusinessLayer/IUserService.cs`) deliberately uses the richer concrete type `UserDto` at its own boundary. If it were removed, `UserDto` would lose its interface, `UserValidator` (`Dto/Validation/UserValidator.cs`) would have nothing to validate against, and `IAddUserViewModel` would lose the `FirstName`/`LastName` declarations its implementations inherit.

### Relationships

- `Dto/UserDto.cs` — `UserDto` implements `IUser` and adds `Id`. It is the object `IUserService.GetActiveUsers()` returns and `IUserService.AddUser()` accepts, but it does so through `UserDto`, not `IUser`.
- `Dto/Validation/UserValidator.cs` — a FluentValidation `AbstractValidator<IUser>` that requires `FirstName` and `LastName` to be non-empty. Validation therefore operates on the interface shape, not the DTO.
- `UserManagement/ViewModel/IAddUserViewModel.cs` — extends `IUser`, so `AddUserViewModel` (`UserManagement/ViewModel/AddUserViewModel.cs`) inherits the `FirstName`/`LastName` properties and derives from `ValidatableViewModelBase<IUser>(new UserValidator())`. This is the main way the interface reaches the UI: the view model *is* an `IUser`.

### Behavior & Details

There is no behaviour to describe beyond the two properties — `FirstName` and `LastName`, each a get/set `string`. Two things are worth knowing:

- The property set is narrower than `UserDto` on purpose. `IUser` omits `Id`, since the add-user form has no identity to show. The consequence is that any code accepting an `IUser` cannot read the user's `Id` without casting back to `UserDto`.
- The interface itself imposes no nullability; the only enforced constraint is the non-empty rule in `UserValidator`, which `AddUserViewModel` applies.

### Rough edges

- The name-based split between `IUser` (view-model side) and `UserDto` (service side) for the same logical entity means a user object must be reshaped at the layer boundary, and `Id` is invisible through the interface.
- `IUserService` uses `UserDto` while everything else in the `Dto` project and the UI validates against `IUser`, so the interface and the service boundary do not share a common user type.