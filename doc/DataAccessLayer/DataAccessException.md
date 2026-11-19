### Purpose

`DataAccessLayer/DataAccessException.cs` defines `DataAccessException`, a custom exception type used to wrap low-level persistence failures into a single, recognizable error that upper layers of the application can catch. It exists to give the `DataAccessLayer` a dedicated error boundary: instead of leaking framework-specific exceptions (e.g., NHibernate internals) to callers, the data-access code rethrows as this type, so consumers know the failure originated in the data-access layer without needing to know which persistence technology is underneath.

### Role in the Codebase

`DataAccessException` is a small shared contract type in the `DataAccessLayer` project, the thin abstraction layer that otherwise holds the persistence interfaces (`IRepository.cs`, `IRepositoryFactory.cs`, `IUnitOfWork.cs`) and entity types (`IEntity.cs`, `Entity.cs`). It is the error side of the data-access contract. Removing it would leave the data-access layer with no typed failure mode of its own; consumers would have to catch generic `Exception` or framework-specific types to distinguish persistence failures.

### Relationships

- It derives from `System.Exception` (standard library). Nothing in the class is overridden; the two constructor arguments are passed straight through to the base constructor.
- **Throw site:** the only place in this repository it is thrown is `HibernateRepositoryFactory<TKey>`'s constructor in `DataAccessLayer.Hibernate/HibernateRepositoryFactory.cs`, which wraps any exception raised while building the `SessionFactory` in `new DataAccessException("Error initializing Hibernate repository factory", e)`.
- **Callers:** no code in this repository currently catches `DataAccessException`. It is a one-directional contract at present — thrown by the data-access layer, awaiting consumers (e.g., the `BusinessLayer`) to catch.

### Behavior & Details

The class has exactly one constructor:

```csharp
public DataAccessException(string message, Exception inner) : base(message, inner)
```

- There is **no parameterless constructor and no message-only constructor**. Every throw site must supply both a message and an inner exception, so the underlying failure is always preserved as `InnerException`. If you want to raise a `DataAccessException` without an underlying exception, you currently have to pass a `null` inner exception explicitly.
- It is only ever thrown during Hibernate repository *factory* initialization. No other data-access path in the repository wraps its failures in `DataAccessException` — ordinary query or save operations surface their framework exceptions directly. Given its role as the layer's error boundary, that coverage is narrower than its name suggests; adding new data-access implementations that wrap failures should throw this type for consistency.