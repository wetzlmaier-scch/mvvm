# Dto/IUser.cs

### Purpose

`Dto/IUser.cs` defines `IUser`, a two-property contract for anything that carries a user's first and last name. It exists so the validation and view-model layers can work with "a user" as a plain name-bearing shape without needing the persistence identifier, which only matters once a record is saved.

### Role in the Codebase

`IUser` is the abstract bottom of the `Dto` project's user model. The concrete side is `UserDto` (`Dto/UserDto.cs`), which implements `IUser` and adds `Id`. Nothing here is logic or infrastructure — it is a marker-like data contract that lets UI-side code and validation code avoid depending on `UserDto` (and therefore on the idea of a stored record).

If the interface were removed, three things would lose their common type: `UserDto`'s implementation relationship, `UserValidator`'s generic base, and `IAddUserViewModel`'s inheritance (see below). It would not break the service boundary, which works in terms of `UserDto` directly.

### Relationships

- `Dto/UserDto.cs` — `UserDto` implements `IUser` and is the only concrete implementation in the solution. It is the object the business layer returns and accepts, but it crosses that boundary as `UserDto`, not as `IUser`.
- `Dto/Validation/UserValidator.cs` — a FluentValidation `AbstractValidator<IUser>` requiring `FirstName` and `LastName` to be non-empty. Validation therefore operates on the interface shape, not on `UserDto`.
- `UserManagement/ViewModel/IAddUserViewModel.cs` — extends `IUser`, so `AddUserViewModel` (`UserManagement/ViewModel/AddUserViewModel.cs`) inherits `FirstName`/`LastName` as its bindable properties and derives from `ValidatableViewModelBase<IUser>` with `UserValidator` for property-level validation.

The dependency direction in the `Dto` project is strictly concrete-to-abstract: anything holding an `IUser` can hold a `UserDto`, and the view models hold their data as `IUser` through `IAddUserViewModel`.

### Behavior & Details

`IUser` declares exactly two auto-properties, `FirstName` and `LastName`, both `string`. There are no methods, defaults, or attributes.

The property set is intentionally narrower than `UserDto`'s. Because `Id` is absent, code accepting an `IUser` cannot read or set the persistence identifier — which is the point: the add-user form and its validation concern only the names a user types. The trade-off is that a user object is reshaped at the layer boundary: `AddUserViewModel.SaveUserAsync()` copies the names from the `IUser`-typed view model into a fresh `UserDto` before calling `UserService.AddUser`, and display paths work the other way with `UserDto` values. `Id` is invisible through the interface on both sides.