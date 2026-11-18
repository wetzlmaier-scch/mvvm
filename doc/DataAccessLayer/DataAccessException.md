# DataAccessException.cs

`DataAccessLayer/DataAccessException.cs`

### Purpose

This file defines `DataAccessException`, the exception type the `DataAccessLayer` project uses to surface persistence failures to its consumers. It exists as a unified failure boundary: rather than leaking NHibernate or ADO.NET exceptions out of the data access layer, persistence errors are wrapped in this type so business logic can tell "the database call failed" apart from application bugs without knowing which ORM or driver is underneath.

### Role in the Codebase

`DataAccessException` is a shared contract type in the `DataAccessLayer` project — the small contract-and-abstraction project that otherwise holds the persistence interfaces (`IRepository<TKey>`, `IRepositoryFactory`, `IUnitOfWork`, `IEntity<TKey>`). The class itself carries no logic; its value is purely in being the type. It is declared `public`, so any project referencing `DataAccessLayer` (such as `DataAccessLayer.Hibernate`, which implements those interfaces) can throw or catch it. Removing it would force callers to either handle ORM-specific exception types directly or lose the ability to catch data-layer failures as a single category.

### Relationships

- **Base type:** `System.Exception` (the .NET standard exception base class). Nothing is overridden; only a single constructor is added.
- **Throw site:** the only place it is thrown in this repository is `HibernateRepositoryFactory<TKey>` in `DataAccessLayer.Hibernate/HibernateRepositoryFactory.cs`. The factory's constructor wraps any failure during NHibernate session-factory setup (`BuildSessionFactory`, schema update, assembly loading) in `throw new DataAccessException("Error initializing Hibernate repository factory", e)`.
- **Callers:** no code in the repository currently catches `DataAccessException`. As it stands it is a one-directional contract — thrown by the data access layer, awaiting consumers (e.g. the `BusinessLayer`) that want to catch persistence failures uniformly.

### Behavior & Details

The class has a single constructor:

```csharp
public DataAccessException(string message, Exception inner) : base(message, inner)
```

It forwards both arguments to the base `Exception` constructor, so the original error is preserved in `InnerException` and the stack trace is captured normally. One consequence of the constructor set worth knowing before throwing this type:

- There is **no parameterless constructor and no message-only constructor**. Every throw site must supply a message and an inner exception. If you want to raise a `DataAccessException` without an underlying exception, you cannot — you would need to add that overload to this class first.

### Rough edges

`DataAccessException` is only ever thrown when the Hibernate repository *factory* fails to initialize — no other data access path in the repository wraps its failures in it. Given its purpose as the layer's unified error boundary, that is a narrow coverage today: a failed query or transaction is not wrapped in this type, so a consumer catching it would not actually catch runtime data-access failures, only factory-creation failures.