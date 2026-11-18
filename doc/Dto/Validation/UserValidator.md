### Purpose

`Dto/Validation/UserValidator.cs` defines `UserValidator`, a FluentValidation rule set that encodes the one user-data rule in this codebase: a user's `FirstName` and `LastName` must both be non-empty. It exists so that any view model dealing with user data can enforce this rule through the shared `ValidatableViewModelBase<T>` plumbing instead of each one re-checking the properties by hand.

### Role in the Codebase

This is the only validator in the `Dto/Validation` directory, and the `Dto` project's sole external dependency (FluentValidation) exists for it. It sits at the boundary between the data model (`Dto`) and the presentation layer: the rule is defined once next to the `IUser` contract, and the UI layer consumes it. If it were removed, `AddUserViewModel` (its known consumer) could no longer be constructed with a validator, and first/last-name emptiness would have to be checked manually before `UserService.AddUser` is called.

### Relationships

- **`IUser`** (`Dto/IUser.cs`) — the type being validated. It is a small interface defined in the same `Dto` project with exactly two properties, `FirstName` and `LastName`, so the two rules in the constructor cover the entire contract.
- **`AbstractValidator<T>`** — the base class comes from the external FluentValidation library (referenced as version 12.1.1 in the `Dto` project). Nothing is overridden; the rule set is declared declaratively in the constructor.
- **`AddUserViewModel`** (`UserManagement/ViewModel/AddUserViewModel.cs`) — the known caller. It passes `new UserValidator()` to its primary constructor, forwarded to `ValidatableViewModelBase<IUser>` (`Mvvm/ValidatableViewModelBase.cs`), which holds the `IValidator<IUser>` and runs it via `ValidateAllProperties` / `ValidateProperty`. Because that base implements `INotifyDataErrorInfo`, the messages from this validator surface directly in the UI.

### Behavior & Details

The constructor declares exactly two rules:

- `RuleFor(x => x.FirstName).NotEmpty().WithMessage("Firstname is required.")`
- `RuleFor(x => x.LastName).NotEmpty().WithMessage("Lastname is required.")`

`NotEmpty()` rejects both `null` and the empty string. Since callers invoke FluentValidation's `Validate()` through the base class, the validator itself is stateless and reusable — each `AddUserViewModel` builds its own instance. The base class calls `validator.Validate(obj)` (all properties on save) and `validator.Validate(obj, options => options.IncludeProperties(propertyName))` (single property on change), so the `WithMessage` strings here are exactly what the user sees in error UI.

One rough edge: the error messages read "Firstname is required." and "Lastname is required." — the casing matches the original literal strings, which differ slightly from the property names (`FirstName`/`LastName`), and the wording says "required" for a rule that also rejects whitespace-only or empty input.