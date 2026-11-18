# `UserDto`

**Path:** `Dto/UserDto.cs`

### Purpose

`UserDto` is the plain data transfer object that carries user information across the boundary between the UI and business layers. It exists so both sides can exchange user data without either depending on the persistence entity (`User` in `Entity.Hibernate`) or on the other's types. The `Dto` project is the neutral middle ground in the layering: it holds the shared shape, the `IUser` contract, and the validation rules that operate on it.

### Role in the Codebase

It is the only concrete `IUser` implementation in the solution and the data shape that crosses the `IUserService` boundary in both directions. Removing it would force the business layer to expose the NHibernate-mapped `User` entity directly, collapsing the separation the `Dto` project exists to provide. It lives alongside `IUser` in the root of the `Dto` project, with the validators under `Dto/Validation/`.

### Relationships

- **`IUser`** (`Dto/IUser.cs`) — the interface `UserDto` implements. `IUser` declares only `FirstName` and `LastName`; `UserDto` adds `Id` on top.
- **`UserService` / `IUserService`** (`BusinessLayer/UserService.cs`, `BusinessLayer/IUserService.cs`) — the main consumer. `GetActiveUsers()` projects each persisted `User` into a `new UserDto { Id, FirstName, LastName }`, and `AddUser(UserDto user)` reads the names back out to construct the entity.
- **`AddUserViewModel`** (`UserManagement/ViewModel/AddUserViewModel.cs`) — the UI-side producer. `SaveUserAsync` builds `new UserDto { FirstName = FirstName, LastName = LastName }` from the form and hands it to `UserService.AddUser`. Note `Id` is never set by callers; it is only ever assigned inside the service.
- **`ShowUsersViewModel`** (`UserManagement/ViewModel/ShowUsersViewModel.cs`) — consumes the `List<UserDto>` from `GetActiveUsers()` and copies only `FirstName`/`LastName` into its view-side `UserModel`.
- **`UserViewModelTest`** (`TestProject1/UserViewModelTest.cs`) — constructs and mocks `UserDto` values against the `IUserService` stub.

### Behavior & Details

`UserDto` has no logic — just three auto-properties:

- **`Id`** (`int`) — the persisted identity. Only the service layer writes it (`GetActiveUsers` copies `u.Id`); UI producers leave it at its default `0`.
- **`FirstName`**, **`LastName`** (`required string`) — the display names. The C# `required` modifier makes any object initializer omitting them a compile error. That is the DTO's one hard guarantee at compile time; whitespace is still allowed and is instead rejected at runtime by the `Guard.IsNotNullOrWhiteSpace` checks inside `UserService.AddUser`.

One asymmetry worth knowing: `IUser` deliberately omits `Id`, because the add-user form has no identity to expose. That means code holding a reference typed as `IUser` cannot reach the user's `Id` without casting back to `UserDto`. The service boundary works around this by using `UserDto` (not `IUser`) directly in its signatures, so in practice the interface-typed path never needs the id.