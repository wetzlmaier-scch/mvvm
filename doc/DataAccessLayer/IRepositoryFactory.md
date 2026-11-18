# `DataAccessLayer/IRepositoryFactory.cs`

## Purpose

This interface defines the contract for a component that produces repository instances and can tear them down. It exists so that consumers of the data access layer can obtain repositories through a factory abstraction instead of instantiating them directly, which decouples business-layer code from the mechanics of opening sessions.

## Role in the Codebase

The interface is the top of a small inheritance chain in the `DataAccessLayer` project:

1. `IRepositoryFactory<TRepository, TKey>` (this file) — generic contract.
2. `IHibernateRepositoryFactory<TKey>` in `DataAccessLayer.Hibernate/IHibernateRepositoryFactory.cs` — binds `TRepository` to `IHibernateRepository<TKey>`, so callers only write `IHibernateRepositoryFactory<TKey>`.
3. `IRepositoryFactory` in `BusinessLayer/IRepositoryFactory.cs` — closes `TKey` to `int`, the application's actual key type.

The concrete implementation is `HibernateRepositoryFactory<TKey>` in `DataAccessLayer.Hibernate/HibernateRepositoryFactory.cs`; the composed instance is `RepositoryFactory` in `BusinessLayer/RepositoryFactory.cs`.

If this file were removed, both `IHibernateRepositoryFactory` and the generic binding it depends on would break, and the business layer's factory interface chain would have nothing to inherit from.

## Relationships

- **Extends nothing**; its constraint references `IRepository<TKey>` from `DataAccessLayer/IRepository.cs`, the generic query API (`GetQueryable`, `Single`, `Find`, `GetAll`, etc.) that created repositories implement.
- **Implemented (indirectly) by** `HibernateRepositoryFactory<TKey>` in `DataAccessLayer.Hibernate/HibernateRepositoryFactory.cs`, which provides the `Create()` and `Drop()` implementations. Note that it implements `Create()` through an explicit interface-implementation cast (the line reading `IRepositoryFactory<IHibernateRepository<TKey>, TKey>.Create()`), so this interface's members are what that file is answering to.
- **Consumers:** Business-layer services take the business-layer `IRepositoryFactory` (e.g., `UserService` in `BusinessLayer/UserService.cs` receives it via its constructor) and call `Create()` to get a repository.

## Behavior & Details

- **`Create()`** — returns a new `TRepository`. The concrete implementation opens a stateless NHibernate session per call, so each returned repository is independent and should be disposed by the caller.
- **`Drop()`** — intended to release anything the factory holds. In the only concrete implementation, `HibernateRepositoryFactory.Drop()`, this is currently `throw new NotImplementedException()`; no call site invokes it, so it is effectively a placeholder for future teardown.

Two caveats worth knowing before relying on this contract:

- **The `out` variance on `TRepository` is not actually honored.** `TRepository` is an output position (only a return type), so `out` looks right, but the constraint `where TRepository : IRepository<TKey>` puts it in an *in* position — generic variance in C# requires that no constraint on a covariant parameter reference the parameter, so this combination is not valid C# (the compiler rejects `where T : SomeInterface<out U>` for a covariant `U`). The code as written therefore signals that the variance attribute was likely added by mistake; it has no effect beyond the compiler error it implies.
- **`Drop()` is aspirational.** Since no implementation can actually honor it yet, do not design shutdown paths around calling `Drop()`.

## Rough edges

- The `out` variance modifier on `TRepository` conflicts with the `IRepository<TKey>` constraint and is not valid C# as written.
- `Drop()` has no working implementation anywhere (`NotImplementedException`), making half of the contract unusable.