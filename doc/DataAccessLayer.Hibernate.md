### Purpose

`DataAccessLayer.Hibernate` is the NHibernate-specific implementation of the repository pattern for this codebase. It provides the concrete classes that query a SQL Server database, persist entities via stateless sessions, and manage transaction scopes, while exposing these capabilities through a set of interfaces that hide NHibernate details from the business layer.

The contents belong together because they form a complete ORM bridge: configuration interfaces (`IModelContext`), factory logic to bootstrap the NHibernate session factory, and the repository/unit-of-work classes that perform the actual data access. This directory acts as the adapter between the abstract `DataAccessLayer` contracts and the NHibernate library.

### Software Components

The components in this directory fall into three functional groups:

*   **Configuration & Bootstrapping:** These components handle the one-time setup of the NHibernate environment.
    *   [`IModelContext`](DataAccessLayer.Hibernate/IModelContext.md) and [`ModelContextBase`](DataAccessLayer.Hibernate/ModelContextBase.md) define and implement the contract for providing the connection string and mapping assembly names.
    *   [`HibernateRepositoryFactory`](DataAccessLayer.Hibernate/HibernateRepositoryFactory.md) is the active bootstrap point. It reads the configuration, builds the `ISessionFactory`, and mints repository instances.
    *   [`SessionFactoryBuilder`](DataAccessLayer.Hibernate/SessionFactoryBuilder.md) contains parallel, unused logic for building the session factory. It is currently dead code, as its `BuildSessionFactory` method is not called by any other component in the repository.

*   **Data Access Implementation:** These are the workhorses that interact with the database.
    *   [`HibernateRepository`](DataAccessLayer.Hibernate/HibernateRepository.md) is the concrete implementation of the repository pattern. It wraps an `IStatelessSession` to provide LINQ query capabilities and an `Insert`-only save method.
    *   [`HibernateUnitOfWork`](DataAccessLayer.Hibernate/HibernateUnitOfWork.md) manages the transaction lifecycle (begin, commit, rollback) for a specific session. It is instantiated lazily by `HibernateRepository` when a transaction is needed.

*   **Abstraction Contracts:** These interfaces allow the business layer to depend on the data layer without referencing NHibernate types directly.
    *   [`IHibernateRepository`](DataAccessLayer.Hibernate/IHibernateRepository.md) extends the base `IRepository` to add save and unit-of-work capabilities.
    *   [`IHibernateRepositoryFactory`](DataAccessLayer.Hibernate/IHibernateRepositoryFactory.md) defines the contract for creating `IHibernateRepository` instances.
    *   [`IHibernateUnitOfWork`](DataAccessLayer.Hibernate/IHibernateUnitOfWork.md) extends the base `IUnitOfWork` to allow specifying transaction isolation levels.

### Relationships

The primary flow of dependency starts at the configuration seam. The business layer provides an `IModelContext` implementation to [`HibernateRepositoryFactory`](DataAccessLayer.Hibernate/HibernateRepositoryFactory.md), which uses those values to initialize the NHibernate `ISessionFactory`. Once initialized, the factory’s `Create` method generates new [`HibernateRepository`](DataAccessLayer.Hibernate/HibernateRepository.md) instances.

Each `HibernateRepository` owns a stateless session. If a transaction is required, the repository lazily creates a [`HibernateUnitOfWork`](DataAccessLayer.Hibernate/HibernateUnitOfWork.md) bound to that same session. The `UnitOfWork` handles the transaction state, while the `Repository` handles the query and insert commands.

[`SessionFactoryBuilder`](DataAccessLayer.Hibernate/SessionFactoryBuilder.md) is isolated from this active flow. Although it shares the same inputs (`connectionString`, `mappingAssemblies`) as the factory, no other component calls it. The logic to build the session factory is duplicated between `SessionFactoryBuilder` and `HibernateRepositoryFactory`, but only the factory’s version is executed in the running application.

```mermaid
flowchart TD
    subgraph "Configuration"
        IModelContext[IModelContext]
        ModelContextBase[ModelContextBase]
    end

    subgraph "Bootstrapping"
        HibernateRepositoryFactory[HibernateRepositoryFactory]
        SessionFactoryBuilder[SessionFactoryBuilder]
    end

    subgraph "Data Access"
        HibernateRepository[HibernateRepository]
        HibernateUnitOfWork[HibernateUnitOfWork]
    end

    subgraph "Contracts"
        IHibernateRepository[IHibernateRepository]
        IHibernateRepositoryFactory[IHibernateRepositoryFactory]
        IHibernateUnitOfWork[IHibernateUnitOfWork]
    end

    ModelContextBase --|implements| IModelContext
    HibernateRepositoryFactory --|implements| IHibernateRepositoryFactory
    HibernateRepository --|implements| IHibernateRepository
    HibernateUnitOfWork --|implements| IHibernateUnitOfWork

    HibernateRepositoryFactory --|creates| HibernateRepository
    HibernateRepository --|lazily creates| HibernateUnitOfWork

    style SessionFactoryBuilder stroke-dasharray: 5 5
    style SessionFactoryBuilder color:#f9f9f9
```

### Rough edges

*   **Duplicated Bootstrapping Logic:** The NHibernate configuration logic exists in both [`HibernateRepositoryFactory`](DataAccessLayer.Hibernate/HibernateRepositoryFactory.md) and [`SessionFactoryBuilder`](DataAccessLayer.Hibernate/SessionFactoryBuilder.md). Only the factory is used, but the duplication creates a risk of divergence if one is updated without the other.
*   **Session Lifecycle Gap:** [`HibernateRepository`](DataAccessLayer.Hibernate/HibernateRepository.md) and [`HibernateUnitOfWork`](DataAccessLayer.Hibernate/HibernateUnitOfWork.md) do not dispose the underlying `IStatelessSession`. Callers must manage the session's lifetime externally, which is not clearly documented in the interface contracts.
*   **Unimplemented Teardown:** The `Drop` method in [`HibernateRepositoryFactory`](DataAccessLayer.Hibernate/HibernateRepositoryFactory.md) throws `NotImplementedException`, meaning there is no supported way to dispose or reconfigure the session factory once it is built.

## Contents
- [HibernateRepository.cs](DataAccessLayer.Hibernate/HibernateRepository.md)
- [HibernateRepositoryFactory.cs](DataAccessLayer.Hibernate/HibernateRepositoryFactory.md)
- [HibernateUnitOfWork.cs](DataAccessLayer.Hibernate/HibernateUnitOfWork.md)
- [IHibernateRepository.cs](DataAccessLayer.Hibernate/IHibernateRepository.md)
- [IHibernateRepositoryFactory.cs](DataAccessLayer.Hibernate/IHibernateRepositoryFactory.md)
- [IHibernateUnitOfWork.cs](DataAccessLayer.Hibernate/IHibernateUnitOfWork.md)
- [IModelContext.cs](DataAccessLayer.Hibernate/IModelContext.md)
- [ModelContextBase.cs](DataAccessLayer.Hibernate/ModelContextBase.md)
- [SessionFactoryBuilder.cs](DataAccessLayer.Hibernate/SessionFactoryBuilder.md)