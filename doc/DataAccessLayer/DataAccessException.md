# DataAccessException.cs

`DataAccessLayer/DataAccessException.cs`

### Purpose

This file defines `DataAccessException`, the exception type the data access layer uses to surface persistence failures. It exists so that consumers of the data access layer can distinguish "something went wrong talking to the data store" from any other kind of exception, without having to inspect underlying driver or ORM exceptions.

### Role in the Codebase

`DataAccessException` is the shared failure signal of the `DataAccessLayer` project. It is a thin marker exception with no logic of its own — its value is purely in the type. Callers that depend on data access can `catch (DataAccessException)` to handle any persistence-layer failure uniformly, regardless of which repository or provider produced it. If the class were removed, that catch point would disappear and callers would have to handle raw ORM/driver exceptions instead.

### Relationships

- `DataAccessException` inherits from `System.Exception` (the standard .NET exception base class); nothing beyond the constructor is overridden.
- It currently has one throw site in the repository: `HibernateRepositoryFactory<TKey>` in `DataAccessLayer.Hibernate/HibernateRepositoryFactory.cs` wraps any failure while building the Hibernate session factory with `throw new DataAccessException("Error initializing Hibernate repository factory", e)`, preserving the original failure as the inner exception.
- No code in the repository currently catches `DataAccessException`; today it is a one-directional contract — thrown by the data access layer, awaiting a consumer.

### Behavior & Details

The class has a single constructor, `DataAccessException(string message, Exception inner)`, which forwards both arguments straight to the base `Exception` constructor. Two consequences follow:

- **There is no parameterless constructor and no message-only constructor.** Callers must always supply a message and an inner exception. Code that wants to throw `DataAccessException` without an underlying cause (e.g. `throw new DataAccessException("msg")`) will not compile — adding such a constructor is a deliberate extension point if that ever becomes needed.
- **The original failure is preserved.** Because the second argument is always passed as the inner exception, the stack trace and message of the root cause (e.g. a SQL or NHibernate error) remain accessible via `InnerException` after the exception propagates up.

The class has no properties, state, or other members; serializing or inspecting it behaves exactly like a plain `Exception`.

Rough edges: the only constructor forces an inner exception to be present, which is fine for wrapping but makes the type awkward to use for purely programmatic data-access failures (validation, misconfiguration) where no underlying exception exists.