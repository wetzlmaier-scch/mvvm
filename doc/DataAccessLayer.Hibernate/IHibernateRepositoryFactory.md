# DataAccessLayer.Hibernate/IHibernateRepositoryFactory.cs

### Purpose

This file defines `IHibernateRepositoryFactory<TKey>`, the contract for a factory that produces NHibernate-backed repositories. It is a thin specialization of the generic `IRepositoryFactory<TRepository, TKey>` from `DataAccessLayer`: it fixes `TRepository` to `IHibernateRepository<TKey>` and re-declares `Create()` as returning `IHibernateRepository<TKey>` directly, so callers write `IHibernateRepositoryFactory<TKey>` instead of the two-parameter generic name.

### Role in the Codebase

It is one of the seam interfaces in the `DataAccessLayer.Hibernate` folder, alongside `IHibernateRepository`, `IHibernateUnitOfWork`, and `IModelContext`. Its role is to let the business layer depend on repository creation without naming any concrete NHibernate types. The only implementation in the repository is `HibernateRepositoryFactory<TKey>` (same folder), which builds an `ISessionFactory` once and hands out `HibernateRepository<TKey>` instances. If this interface were removed, `HibernateRepositoryFactory` and the whole business-layer factory chain below it would lose their contract.

### Relationships

- **Extends** `IRepositoryFactory<IHibernateRepository<TKey>, TKey>` (defined in `DataAccessLayer/IRepositoryFactory.cs`), which declares `TRepository Create()` and `void Drop()` and constrains `TRepository : IRepository<TKey>` and `TKey : struct`.
- **Produces** `IHibernateRepository<TKey>` (defined in `DataAccessLayer.Hibernate/IHibernateRepository.cs`), the repository interface with `Save<TEntity>` and the `UnitOfWork` property.
- **Implemented by** `HibernateRepositoryFactory<TKey>` (`DataAccessLayer.Hibernate/HibernateRepositoryFactory.cs`), the single concrete factory.
- **Consumed by the business layer**: `BusinessLayer/IRepositoryFactory.cs` is an empty non-generic interface extending `IHibernateRepositoryFactory<int>`, and `BusinessLayer/RepositoryFactory.cs` (`[Export]`-ed as MEF part) inherits `HibernateRepositoryFactory<int>`. So the practical flow is: business code resolves `BusinessLayer.IRepositoryFactory`, calls `Create()`, and receives an `IHibernateRepository<int>`.

### Behavior & Details

The interface is minimal, but one detail is worth knowing because it is easy to trip over:

- `new IHibernateRepository<TKey> Create();` — the `new` keyword hides the base `IRepositoryFactory<TRepository, TKey>.Create()` rather than overriding it (interfaces cannot override). A caller holding `IHibernateRepositoryFactory<TKey>` gets the `IHibernateRepository<TKey>` return type, which is the point of the re-declaration. The base declaration is still visible when the same object is viewed through `IRepositoryFactory<IHibernateRepository<TKey>, TKey>`; `HibernateRepositoryFactory<TKey>` implements the explicit interface method `IRepositoryFactory<IHibernateRepository<TKey>, TKey>.Create()` by delegating to the public `Create()`, so both views return the same repository.
- `void Drop()` is inherited unchanged from the base. In the only implementation it throws `NotImplementedException`, so do not rely on `Drop()` to release anything today.
- The `where TKey : struct` constraint is repeated from the base and is what allows `TKey` to be `int` in the business layer's `IHibernateRepositoryFactory<int>` specialization.

### Rough edges

- The `new` modifier here is a code smell in its own right: re-declaring an inherited interface member with `new` is a classic sign that the base interface's type parameter should have been constrained more specifically. It works, but callers that upcast to the base interface get the less specific return type.