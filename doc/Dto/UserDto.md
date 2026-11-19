# Dto/UserDto.cs

### Purpose

`UserDto` is the plain data transfer object that carries user information across the boundary between the `UserManagement` UI layer and the `BusinessLayer`. It exists so both sides can exchange user data without either depending on the NHibernate-persisted entity or on the other's types. It holds exactly three auto-properties: `Id`, `FirstName`, and `LastName`, and nothing else — no logic, no behaviour.

### Role in the Codebase

The class is the only concrete `IUser` implementation in the solution and the data shape that crosses the `IUserService` boundary in both directions: `GetActiveUsers()` returns `Task<List<UserDto>>` and `AddUser()` takes a `UserDto`. It lives at the bottom of the `Dto` project's layering, alongside `IUser` (`Dto/IUser.cs`) and the FluentValidation rules in `Dto/Validation/`.

It is the object the business layer produces and the object the view models consume, so removing it would force the service contract to expose either the persistence entity or its own type — breaking the current clean separation.

### Relationships

- **`IUser`** (`Dto/IUser.cs`) — the interface `UserDto` implements. `IUser` declares only `FirstName` and `LastName`; `UserDto` adds `Id` on top. This split is deliberate: `Id` is the persistence identifier the service layer needs but the UI's add-user form does not, and `IUser` is the shape the UI-side validation (`UserValidator`) and view models program against. Any code holding a reference typed as `IUser` cannot reach the user's `Id` without an explicit cast to `UserDto`.
- **`UserService` / `IUserService`** (`BusinessLayer/UserService.cs`, `BusinessLayer/IUserService.cs`) — the main consumer. `GetActiveUsers()` projects each persisted `User` into a `new UserDto { Id = ..., FirstName = ..., LastName = ... }`, and `AddUser(UserDto)` reads the name fields to persist a new user.
- **`AddUserViewModel`** (`UserManagement/ViewModel/AddUserViewModel.cs`) — the UI-side producer. `SaveUserAsync` builds `new UserDto { FirstName = FirstName, LastName = LastName }` from the form fields (leaving `Id` unset) and hands it to `UserService.AddUser`.
- **`ShowUsersViewModel`** (`UserManagement/ViewModel/ShowUsersViewModel.cs`) — consumes the `List<UserDto>` returned by `GetActiveUsers()` and copies only `FirstName`/`LastName` into its view-side `UserModel`.
- **`UserViewModelTest`** (`TestProject1/UserViewModelTest.cs`) — constructs and mocks `UserDto` values against the `IUserService` stub.

### Behavior & Details

`UserDto` has no logic — just three auto-properties:

| Property | Type | Notes |
|---|---|---|
| `Id` | `int` | Persistence identifier. Not part of `IUser`; only populated by the service layer when projecting from a persisted entity. |
| `FirstName` | `string` | Marked `required` (C# required member), so object initializers that omit it do not compile. |
| `LastName` | `string` | Marked `required` for the same reason. |

One asymmetry worth knowing: `IUser` deliberately omits `Id` because the add-user form has no identity to expose. The consequence is that a `UserDto` is a superset of `IUser` — any `IUser` reference can hold a `UserDto`, but code holding `IUser` cannot read `Id` without an explicit cast.

### Rough edges

The `required` modifier on `FirstName`/`LastName` is enforced at compile time only for object initializers; code that constructs a `UserDto` via a memberwise constructor (which exists implicitly) or reflection can still leave the fields unset. In practice, every construction site in the codebase (the service's projection and `AddUserViewModel.SaveUserAsync`) supplies both names, so this is a latent, not active, concern.