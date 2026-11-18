# DataAccessLayer.Hibernate/IHibernateRepositoryFactory.cs

### Purpose

This file defines `IHibernateRepositoryFactory<TKey>`, the contract for a factory that produces Hibernate-based repositories. It exists to give callers a stable interface for requesting NHibernate-backed repositories while hiding the session factory and configuration details inside the concrete implementation.

### Role in the Codebase

It is the Hibernate-specific factory contract inside the `DataAccessLayer.Hibernate` namespace. The generic, engine-agnostic contract is `IRepositoryFactory<TRepository, TKey>` in `DataAccessLayer/IRepositoryFactory.cs`, which declares `Create()` and `Drop()` and constrains `TRepository` to `IRepository<TKey>`. This interface specializes that contract to the Hibernate repository type, so the business layer can ask for a factory without depending on any concrete data-access implementation.

### Relationships

- **`IRepositoryFactory<TRepository, TKey>`** (`DataAccessLayer/IRepositoryFactory.cs`): the base interface. It declares `Create()` and `void Drop()`. This interface inherits both; `Drop()` carries over unchanged, and `Create()` is hidden via `new` with a more specific return type of `IHibernateRepository<TKey>` instead of the base `TRepository` type argument.
- **`IHibernateRepository<TKey>`** (`DataAccessLayer.Hibernate/IHibernateRepository.cs`): the repository type returned by `Create()`. It extends `IRepository<TKey>` and `IDisposable` and adds `Save<TEntity>` and a `UnitOfWork` property.
- **`HibernateRepositoryFactory<TKey>`** (`DataAccessLayer.Hibernate/HibernateRepositoryFactory.cs`): the concrete implementation. It builds the NHibernate `ISessionFactory` in its constructor, and its `Create()` returns a `HibernateRepository<TKey>` wrapping a stateless session. Because `TRepository` is an out-variance (`out TRepository`) position, that implementation satisfies this interface.
- **`BusinessLayer.IRepositoryFactory`** (`BusinessLayer/IRepositoryFactory.cs`): a non-generic marker interface that simply extends `IHibernateRepositoryFactory<int>`. This is the form the business layer would actually hold and resolve, so this interface is what business-layer code indirectly depends on.

### Behavior & Details

- `TKey` is constrained to `struct`, matching the base `IRepositoryFactory`.
- The `new IHibernateRepository<TKey> Create()` declaration **hides** rather than overrides the base `Create()` (interfaces cannot override). A caller holding an `IHibernateRepositoryFactory<TKey>` reference gets the specific `IHibernateRepository<TKey>` return type; a caller holding only the base `IRepositoryFactory<IHibernateRepository<TKey>, TKey>` reference gets the base return type and must cast if it wants the Hibernate-specific members like `Save` or `UnitOfWork`. The concrete class bridges the two by providing an explicit implementation of the base `Create()` that simply delegates to the public one.