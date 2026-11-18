# UserDto

**Path:** `Dto/UserDto.cs`

### Purpose

`UserDto` is the plain data object (DTO) that carries user information across the project's layer boundary. It exists so the UI and business layers exchange user data without depending on the persistence layer's `User` entity directly.

### Role in the Codebase

It is a data model in the `Dto` namespace, sitting alongside its contract, `IUser` (`Dto/IUser.cs`). It has no logic of its own — no methods, no validation — and is constructed, read, and passed around by other layers.

### Relationships

- **Implements `IUser`** (`Dto/IUser.cs`), which declares `FirstName` and `LastName`. The `Id` property here is an addition beyond the interface — the interface only captures the name fields.
- **Consumed by the business layer:** `IUserService.GetActiveUsers()` (`BusinessLayer/IUserService.cs`) returns `List<UserDto>`, and `AddUser(UserDto user)` accepts one. Its implementation, `UserService`, maps the NHibernate `User` entity into a `UserDto` when reading, and reads `FirstName`/`LastName` back out of it when writing.
- **Produced by the UI layer:** `AddUserViewModel` (`UserManagement/ViewModel/AddUserViewModel.cs`) builds a `UserDto` from form input and passes it to `UserService.AddUser`.
- **Used in tests:** `UserViewModelTest` (`TestProject1/UserViewModelTest.cs`) constructs and mocks `UserDto` values.

### Behavior & Details

- `Id` is an `int`, set by the persistence layer (populated when `UserService` maps an entity in `GetActiveUsers`). Callers should treat it as read-only — `AddUser` ignores it when creating the entity.
- `FirstName` and `LastName` are `required` properties, enforced at compile time by C#'s required-members feature: any object initializer for `UserDto` that omits them fails to compile. This is the only guardrail in the class itself.
- Note that `required` does not reject empty or whitespace strings at runtime; the actual non-blank validation happens later in `UserService.AddUser`, which uses `Guard.IsNotNullOrWhiteSpace` on both fields before saving.

### Rough edges

`UserDto` implements `IUser`, which only declares `FirstName` and `LastName`, so the `Id` property is invisible through the interface. Any code taking an `IUser` (e.g. to display or persist a user) has no way to get the `Id` without casting back to `UserDto`.