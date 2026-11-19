# Dto/Validation/UserValidator.cs

### Purpose

`UserValidator` is the FluentValidation rule set that encodes the application's only user-data rule: a user's `FirstName` and `LastName` must both be non-empty. It exists so that validation of user data lives in one place, in the `Dto` project, rather than being scattered across view models or UI code.

### Role in the Codebase

It is the single component in the `Dto/Validation/` directory and the only FluentValidation validator in this codebase. It is a plain rule-set class with no state — the rules are fixed and fully defined in the constructor. Nothing else in the `Dto` project depends on it; it is a leaf that UI-layer view models consume. If it were removed, `AddUserViewModel` (the known caller) could no longer enforce non-empty names, since the `IUser` interface itself imposes no nullability or other constraint.

### Relationships

- **Base type:** `AbstractValidator<IUser>` from the external FluentValidation library. The class inherits the validation machinery and supplies only the rules.
- **Validation target:** `IUser`, defined in `Dto/IUser.cs`. The interface exposes exactly `FirstName` and `LastName`, and this validator targets precisely those two properties — validation operates on the interface shape, so any `IUser` implementation is validated identically.
- **Caller:** `AddUserViewModel` (`UserManagement/ViewModel/AddUserViewModel.cs`) constructs `new UserValidator()` and passes it to `ValidatableViewModelBase<IUser>` (`Mvvm/ValidatableViewModelBase.cs`), whose `ValidateAllProperties`/`ValidateProperty` methods run the validator and surface `INotifyDataErrorInfo` errors to the UI. `UserViewModelTest` in `TestProject1` exercises the behaviour end-to-end through that view model.

### Behavior & Details

The constructor registers two rules:

- `RuleFor(x => x.FirstName).NotEmpty().WithMessage("Firstname is required.")`
- `RuleFor(x => x.LastName).NotEmpty().WithMessage("Lastname is required.")`

`NotEmpty()` fails on both `null` and empty strings (`string.Empty`), but **not** on whitespace-only strings. A user typed as `"   "` passes this validator. Error messages are the literals above, and they reach the UI through `ValidatableViewModelBase`'s `GetErrors`/`HasErrors`.

### Rough edges

- The `NotEmpty` rule means whitespace-only names are valid per the validator, yet `TestProject1/UserViewModelTest` includes whitespace cases that assert rejection. Those assertions pass only if a trim or whitespace rule lives somewhere not visible in this file (none is present here), so the test and this validator do not line up as written.
- The message spelling is inconsistent with the property names: "Firstname"/"Lastname" (lowercase "name") versus `FirstName`/`LastName`.