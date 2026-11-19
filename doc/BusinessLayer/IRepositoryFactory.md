# `BusinessLayer/IRepositoryFactory.cs`

### Purpose
This file defines `IRepositoryFactory`, the business layer's non-generic contract for creating NHibernate-backed repositories. It exists so that business services can depend on a factory type with the key already closed to `int`, the application's entity key type. Without it, each service would have to name the generic `IHibernateRepositoryFactory<int>` directly or reach into the `DataAccessLayer` project's abstractions.

The interface body is empty. It adds no members of its own; it simply extends `IHibernateRepositoryFactory<int>`, which itself extends `DataAccessLayer.IRepositoryFactory<IHibernateRepository<TKey>, TKey>`. So everything a caller can do through `IRepositoryFactory` is inherited: `Create()` returns an `IHibernateRepository<int>`, and `Drop()` is the (unimplemented) teardown method from the data access layer contract.

### Role in the Codebase
It is a type alias, in effect: a non-generic handle to a generic hierarchy. It sits in `BusinessLayer` next to its sole implementation, `RepositoryFactory` (`BusinessLayer/RepositoryFactory.cs`). `RepositoryFactory` extends `HibernateRepositoryFactory<int>` and publishes itself to MEF with `[Export(typeof(IRepositoryFactory))]`. MEF composition in the host applications resolves `IRepositoryFactory` to that concrete class, so business services never touch NHibernate types or the generic factory names.

Removing this interface would not remove capability — callers could depend on `IHibernateRepositoryFactory<int>` directly — but it would drop the project's stable, engine-neutral name for the factory inside `BusinessLayer` and force every service's constructor and field to carry the `DataAccessLayer.Hibernate` generic type instead.

### Relationships
- **Extends** `DataAccessLayer.Hibernate.IHibernateRepositoryFactory<TKey>` (defined in `DataAccessLayer.Hibernate/IHibernateRepositoryFactory.cs`), closed at `TKey = int`. Through it, the contract inherits from `DataAccessLayer.IRepositoryFactory<TRepository, TKey>` (`DataAccessLayer/IRepositoryFactory.cs`), which declares `TRepository Create()` and `void Drop()`.
- **Implemented by** `RepositoryFactory` (`BusinessLayer/RepositoryFactory.cs`), the MEF-exported concrete class.
- **Consumed by** `UserService` (`BusinessLayer/UserService.cs`), which takes `IRepositoryFactory` in its `[ImportingConstructor]` and calls `_repositoryFactory.Create()` inside a `using` block for each operation. Any other future service in `BusinessLayer` that needs repositories would take this same type.

### Behavior & Details
There is no logic to describe — an empty interface contributes only its base contract. Two things worth knowing about what falls out of the inheritance chain:

- `IHibernateRepositoryFactory<TKey>.Create()` is declared with the `new` keyword, so it hides the base `IRepositoryFactory<TRepository, TKey>.Create()` rather than overriding it. A caller holding `IRepositoryFactory` resolves `Create()` to the hidden `IHibernateRepository<int>`-returning overload (the most-derived applicable member), which is what services actually want: a repository exposing `GetQueryable<T>()`, `Save(...)`, and a `UnitOfWork`.
- `Drop()` is inherited from `DataAccessLayer.IRepositoryFactory<TRepository, TKey>` but no business-layer caller uses it, and the concrete `HibernateRepositoryFactory` implements it by throwing `NotImplementedException`. The practical teardown path is the `using` disposes in services, which dispose the returned repository, not the factory.

### Rough edges
- The name `IRepositoryFactory` shadows `DataAccessLayer.IRepositoryFactory<TRepository, TKey>` in any file that imports both `BusinessLayer` and `DataAccessLayer`. Code that needs both must qualify by namespace.
- Because the interface is empty and its only implementation is a MEF export, the business layer's "engine-neutral" seam is only one hop away from NHibernate: `Create()` returns an `IHibernateRepository<int>`, an NHibernate-specific type, so services still transitively depend on the `DataAccessLayer.Hibernate` abstractions.