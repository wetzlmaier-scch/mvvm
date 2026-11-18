# Dto/Validation/UserValidator.cs

### Purpose

`UserValidator` is a FluentValidation validator for `IUser` (`Dto/IUser.cs`). It encodes the one business rule the user-management feature has for user data: both the first name and the last name must be non-empty. It exists so that UI viewmodels can check user input against a shared, reusable rule set instead of hand-rolling checks.

### Role in the Codebase

It lives in the `Dto` assembly under `Validation/`, next to the `IUser` and `UserDto` types it validates. It is a leaf helper with no other responsibilities — no entry point, no state, no side effects. Nothing breaks at compile time if it is removed, but the viewmodels that construct it (see below) lose their input validation.

### Relationships

- **Base type:** `AbstractValidator<T>` from the FluentValidation library (external, third-party). Inheriting it is all that's required to make this validator usable through `IValidator<IUser>`.
- **Validated type:** `IUser` from `Dto/IUser.cs`, which defines exactly two properties: `FirstName` and `LastName`. The class also implements it, via `UserDto` in `Dto/UserDto.cs`, so the validator applies to both.
- **Known caller:** `AddUserViewModel` in `UserManagement/ViewModel/AddUserViewModel.cs` constructs `new UserValidator()` and passes it to its base class `ValidatableViewModelBase<T>` (`Mvvm/ValidatableViewModelBase.cs`), which runs `ValidateAllProperties`/`ValidateProperty` through the injected `IValidator<T>`. Any other viewmodel can do the same.

### Behavior & Details

The constructor declares two rules:

- `RuleFor(x => x.FirstName).NotEmpty().WithMessage("Firstname is required.")`
- `RuleFor(x => x.LastName).NotEmpty().WithMessage("Lastname is required.")`

Both use FluentValidation's `NotEmpty` rule, which fails on `null`, empty strings, and whitespace-only strings — so a blank or space-only field produces the "Firstname/Lastname is required." error message.

In practice the validator never validates `IUser` directly. `ValidatableViewModelBase.ValidateAllProperties` and `ValidateProperty` call `validator.Validate(obj)` on whatever object the viewmodel passes in. `AddUserViewModel` passes `this` (itself an `IUser`, which its base class implements), so the validated `FirstName`/`LastName` come from the viewmodel's observable properties, not from a `UserDto` instance. The `ValidateProperty` overload also relies on FluentValidation's `IncludeProperties` option, so the property names in the rules must match the viewmodel's property names — `FirstName` and `LastName` do.

### Rough edges

- The rules only cover `FirstName` and `LastName`, but the base class's `HasErrors` property is `Count != 0` over the whole dictionary of property names — combined with the base's `ValidateAllProperties` clearing it first, that's consistent, but note `HasErrors` alone (without a prior `ValidateAllProperties`) would report stale errors from a previous `ValidateProperty` call for a different property. That quirk belongs to `ValidatableViewModelBase` rather than this validator, and is worth knowing when debugging unexpected `HasErrors` results.