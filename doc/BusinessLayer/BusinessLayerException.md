# `BusinessLayer/BusinessLayerException.cs`

### Purpose

This file defines `BusinessLayerException`, a minimal `Exception` subclass that marks "a business operation in the `BusinessLayer` project failed." It exists so code above the business layer can distinguish business failures from other exceptions — the codebase treats this type as the error contract of the `BusinessLayer` project. A comment in `UserService.AddUser` states the intent: the save path should "only throw a BusinessException".

### Role in the Codebase

The class is a marker/contract type, not an architectural component. It contains no state, no properties, and a single constructor that forwards to `Exception(string, Exception)`. If it were removed, `UserService.AddUser` and its tests would no longer compile, and callers could no longer catch business-layer failures by a dedicated type.

### Relationships

- **Parent type:** `System.Exception` — standard BCL. Everything is inherited; nothing is overridden.
- **Thrown from:** `UserService.AddUser` in `BusinessLayer/UserService.cs` (line 55): after rolling back the NHibernate transaction and logging via `LoggingService`, it rethrows the underlying failure as `new BusinessLayerException("Error saving user", e)`. The `inner` parameter is where data-access-level detail is preserved.
- **Caught by:** `AddUserViewModel.SaveUserAsync` in `UserManagement/ViewModel/AddUserViewModel.cs` — it has a `catch (BusinessLayerException) { throw; }` block, a bare rethrow.
- **Constructed in tests:** `TestProject1/UserViewModelTest.cs` — `SaveUserAsync_WhenServiceThrows_RethrowsExceptionAndDoesNotAddUser` builds one with a fake inner `Exception`, stubs `IUserService.AddUser` to throw it, and asserts the exception propagates unchanged (`Assert.ThrowsExactlyAsync<BusinessLayerException>`).

### Behavior & Details

- Single constructor: `BusinessLayerException(string message, Exception inner)`, forwarded to `base(message, inner)`. There is no parameterless constructor and no other overloads, so it can only be created with both a message and an inner exception — which matches how `UserService` uses it, wrapping the original failure rather than discarding it.
- No custom state or behavior: `Message`, `InnerException`, `StackTrace`, serialization, and all other members behave exactly like `Exception`.

### Rough edges

- `AddUserViewModel` catches `BusinessLayerException` only to rethrow it with `throw;`, so the dedicated catch has no observable effect at present; the separating purpose of the type is exercised by the tests, not by any handler logic.
- `UserService.AddUser` validates input with `Guard` before the `try` block, and `Guard` throws plain `ArgumentException`. Validation failures therefore never become a `BusinessLayerException`, which undercuts the "only throw a BusinessException" contract the class is meant to express.