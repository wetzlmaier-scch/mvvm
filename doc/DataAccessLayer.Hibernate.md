# DataAccessLayer.Hibernate

This directory is a C# class library project (`DataAccessLayer.Hibernate`) that provides the concrete NHibernate implementation for the data access abstractions defined in the sibling `DataAccessLayer` project.

### Purpose

It serves as the persistence engine for the application, specifically targeting SQL Server via NHibernate. Its primary responsibility is to manage the lifecycle of NHibernate's `ISessionFactory`, handle schema auto-updating based on Fluent mappings, and produce repository instances that allow business logic to query and persist entities. The code belongs together because it forms the complete stack required to bridge the application's generic repository interfaces with a specific ORM provider, from the initial session factory bootstrap down to individual transactional units of work.

### Software Components

The components in this directory fall into three main groups based on their lifecycle and responsibility:

*   **Configuration & Context:** `IModelContext` and `ModelContextBase` define the contract for database connection strings and mapping assembly names. `ModelContextBase` is an abstract base class that simply stores these values, ensuring the data access layer does not hard-code configuration.
*   **Factory & Bootstrap:** `HibernateRepositoryFactory<TKey>` is the central entry point. It implements `IHibernateRepositoryFactory<TKey>` and is responsible for the one-time initialization of the NHibernate session factory (including loading mappings and updating the database schema). It uses a static guard to ensure only one session factory exists per process. `SessionFactoryBuilder` is a static utility that contains duplicate logic for building the session factory, but it is currently not used by the application.
*   **Repositories & Transactions:** `HibernateRepository<TKey>` implements `IHibernateRepository<TKey>`. It wraps a stateless NHibernate session and exposes LINQ-to-NHibernate queries and a single `Save` method (insert-only). `HibernateUnitOfWork` implements `IHibernateUnitOfWork` and manages the transaction lifecycle (begin, commit, rollback) for the repository's session.

### Relationships

The dependency flow is strictly one-way, moving from the factory down to the repository and unit of work.

1.  **Bootstrap:** The `BusinessLayer` (outside this directory) provides an `IModelContext` instance to `HibernateRepositoryFactory<TKey>`. The factory uses this context to configure NHibernate, update the schema, and build the `ISessionFactory`.
2.  **Creation:** When business code needs data access, it calls `Create()` on the factory. This method opens a new `IStatelessSession` and wraps it in a `HibernateRepository<TKey>`.
3.  **Operation:** The `HibernateRepository<TKey>` exposes query methods that execute against its stateless session. If a write operation occurs, it uses the `UnitOfWork` property.
4.  **Transaction:** The `UnitOfWork` property lazily instantiates a `HibernateUnitOfWork` bound to the same session. This allows business code to explicitly begin, commit, or roll back transactions around a group of operations.

The interfaces (`IHibernateRepository`, `IHibernateRepositoryFactory`, `IHibernateUnitOfWork`) decouple the business layer from the concrete NHibernate implementations, allowing the business layer to depend only on the abstractions.

```mermaid
flowchart TD
    subgraph "Configuration"
        IModelContext[IModelContext]
        ModelContextBase[ModelContextBase]
    end

    subgraph "Factory & Bootstrap"
        IFactory[IHibernateRepositoryFactory<TKey>]
        Factory[<b>HibernateRepositoryFactory<TKey></b>]
        SessionFactoryBuilder[SessionFactoryBuilder]
    end

    subgraph "Repository & Transaction"
        IRepository[IHibernateRepository<TKey>]
        Repository[<b>HibernateRepository<TKey></b>]
        IUnitOfWork[IHibernateUnitOfWork]
        UnitOfWork[<b>HibernateUnitOfWork</b>]
    end

    IModelContext -->|implemented by| ModelContextBase
    ModelContextBase -->|provides config to| Factory
    IFactory -->|implemented by| Factory
    IUnitOfWork -->|implemented by| UnitOfWork
    IRepository -->|implemented by| Repository
    Factory -->|creates| Repository
    Repository -->|owns session for| UnitOfWork
    Repository -->|exposes| IUnitOfWork
    Factory -.->|duplicates logic (unused)| SessionFactoryBuilder
```

### Rough edges

*   **Duplicated Configuration Logic:** `SessionFactoryBuilder` and the constructor of `HibernateRepositoryFactory<TKey>` both contain logic to configure NHibernate and update the schema. `SessionFactoryBuilder` is currently dead code, but its existence creates a risk of configuration drift if both paths were ever used.
*   **Resource Leaks in Disposal:** `HibernateRepository<TKey>.Dispose()` disposes the `HibernateUnitOfWork` but does not explicitly dispose the underlying `IStatelessSession`. Since the session is opened by the factory and owned by the repository, this can lead to session leaks if callers do not manage disposal carefully.
*   **Schema Mutation on Start:** Both the factory and the builder execute `SchemaUpdate` during construction. This means the database schema is modified every time the application starts, which can be risky in production environments if not strictly controlled.

## Contents
- [DataAccessLayer.Hibernate.csproj](DataAccessLayer.Hibernate/DataAccessLayer.Hibernate.md)
- [HibernateRepository.cs](DataAccessLayer.Hibernate/HibernateRepository.md)
- [HibernateRepositoryFactory.cs](DataAccessLayer.Hibernate/HibernateRepositoryFactory.md)
- [HibernateUnitOfWork.cs](DataAccessLayer.Hibernate/HibernateUnitOfWork.md)
- [IHibernateRepository.cs](DataAccessLayer.Hibernate/IHibernateRepository.md)
- [IHibernateRepositoryFactory.cs](DataAccessLayer.Hibernate/IHibernateRepositoryFactory.md)
- [IHibernateUnitOfWork.cs](DataAccessLayer.Hibernate/IHibernateUnitOfWork.md)
- [IModelContext.cs](DataAccessLayer.Hibernate/IModelContext.md)
- [ModelContextBase.cs](DataAccessLayer.Hibernate/ModelContextBase.md)
- [SessionFactoryBuilder.cs](DataAccessLayer.Hibernate/SessionFactoryBuilder.md)