# `BusinessLayer/BusinessLayerException.cs`

This file defines `BusinessLayerException`, the single exception type used by the `BusinessLayer` project to signal that a business operation failed.

### Purpose

The class exists so business-logic failures are distinguishable from other kinds of errors. It inherits from the .NET `System.Exception` (a framework type, not defined in this repository), and its only addition is its *identity*: catching it tells a caller that the failure originated in the business layer and wrapped an underlying cause, rather than being a validation error, infrastructure fault, or framework exception.

### Role in the Codebase

`BusinessLayerException` is the error contract of the `BusinessLayer` project (`BusinessLayer/BusinessLayerException.cs`). Its main producer is `BusinessLayer/UserService.cs`: when `UserService.AddUser` fails while saving a user, the method rolls back the transaction, logs the original error, and rethrows it wrapped in a `BusinessLayerException` — the code comment there ("only throw a BusinessException") marks this as an intentional rule for that service.

Downstream, `UserManagement/ViewModel/AddUserViewModel.cs` catches `BusinessLayerException` explicitly in its save command. Note, however, that that catch block currently just does `throw;` — it rethrows unchanged, so today it distinguishes nothing yet; it is a placeholder for handling the business-layer failure separately from other exceptions. `TestProject1/UserViewModelTest.cs` constructs the exception directly (`new BusinessLayerException("Database connection failed", new Exception())`) and asserts that `SaveUserCommand` throws exactly this type, so the type is also part of the tested behaviour contract.

Removing this class would break `UserService`, `AddUserViewModel`, and the test project compilation, and more importantly remove the marker that lets callers tell business failures apart from other exceptions.

### Behavior & Details

The class has exactly one constructor:

```csharp
public BusinessLayerException(string message, Exception inner) : base(message, inner)
```

- It requires **both** a message and an inner exception. There is no parameterless or message-only constructor, so every `BusinessLayerException` carries the original fault as `InnerException`. Code that wants to throw it with no cause (as the test does) must explicitly pass one, e.g. `new Exception()`.
- The constructor does nothing but forward both arguments to the `Exception` base constructor. No state is added; there is nothing to serialize or configure beyond the standard exception members.

Because the only constructor demands an `inner`, this exception is exclusively a *wrapper*: it is meant to be thrown around a caught exception, never on its own.

### Rough edges

- `AddUserViewModel` catches `BusinessLayerException` and rethrows it with no additional handling, so the dedicated catch currently has no observable effect and the type's purpose of separating business failures is only realized at the `UserService` boundary.
- The class omits a `SerializationInfo`-based (deserialization) constructor, which is the conventional third constructor on a custom `Exception` subclass; `BusinessLayerException` therefore cannot be recreated by the serializer after crossing an AppDomain/assembly boundary.