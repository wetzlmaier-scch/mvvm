# `BusinessLayer/BusinessLayerException.cs`

### Purpose

This file defines `BusinessLayerException`, a single `Exception` subclass that serves as the `BusinessLayer` project's marker type for "a business operation failed." It exists so that callers outside the layer can distinguish business-layer faults from framework, validation, or data-access errors by catching a specific type instead of a bare `Exception`.

The class has no fields, properties, or logic of its own — it adds nothing to `System.Exception` beyond its identity as a type. All the value of the file is in its name.

### Role in the Codebase

It is the error contract of the `BusinessLayer` project. The code in `BusinessLayer/UserService.cs` carries a comment making this explicit: in `AddUser`, any exception from the save path is logged and rethrown as `throw new BusinessLayerException("Error saving user", e)`, so business failures leave the service wrapped in this one type.

If it were removed, `UserService` would have nothing to wrap faults in, and the catch and tests below would have no type to target.

### Relationships

- **Producer:** `BusinessLayer/UserService.cs` — the only place in production code that throws it. `AddUser`'s `catch (Exception e)` block constructs it with the message `"Error saving user"` and the original fault as the inner exception, after rolling back the transaction and logging via `LoggingService`.
- **Consumer:** `UserManagement/ViewModel/AddUserViewModel.cs` — `SaveUserAsync` catches `BusinessLayerException` explicitly and rethrows it (`throw;`), so business failures propagate to the command's caller.
- **Tests:** `TestProject1/UserViewModelTest.cs` — `SaveUserAsync_WhenServiceThrows_RethrowsExceptionAndDoesNotAddUser` constructs one with a fake inner `Exception`, stubs `IUserService.AddUser` to throw it, and asserts `Assert.ThrowsExactlyAsync<BusinessLayerException>` — note the *exactly*, so the type must not be silently widened to a derived or different exception.

It itself depends on nothing but `System.Exception` (framework type).

### Behavior & Details

One constructor, and it is the only door in:

```csharp
public BusinessLayerException(string message, Exception inner) : base(message, inner)
```

- Both arguments are required. There is no parameterless or message-only constructor, so every instance is guaranteed to carry the original fault in `InnerException`. Code that wants to raise one for a non-wrapped reason still has to pass some inner `Exception` (the test does exactly this with `new Exception()`).
- `message` and `inner` are passed straight through to `Exception`'s `(message, inner)` constructor. Nothing is modified, enriched, or validated beyond what `Exception` does (e.g. `Exception` tolerates a `null` inner exception).
- No `SerializationInfo`-based deserialization constructor, so a serialized instance of this type cannot be deserialized. Irrelevant in practice here since nothing serializes this exception, but it is missing relative to the conventional three-constructor shape for custom `Exception` subclasses.

### Rough edges

- `AddUserViewModel` catches `BusinessLayerException` only to `throw;` with no other handling, so the dedicated catch currently has no observable effect — the type's separating purpose is asserted by the test rather than acted on by the UI layer.
- `UserService.AddUser`'s `Guard` validation calls throw plain `ArgumentException` before the try block, so validation failures bypass the "only throw a BusinessLayerException" intent stated in the code; callers catching this type will not see bad-input failures.