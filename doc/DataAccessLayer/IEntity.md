### Purpose

`DataAccessLayer/IEntity.cs` defines `IEntity<TKey>`, the marker contract for anything the data access layer treats as a persistable entity: a reference type exposing a read-only `Id` whose type is a value type. It exists so that the repository abstractions (`IRepository<TKey>`, `IHibernateRepository<TKey>`) can constrain their generic entity parameters to "has an identity" without depending on any concrete base class.

### Role in the Codebase

`DataAccessLayer` is a small contracts-only project — persistence interfaces (`IRepository.cs`, `IRepositoryFactory.cs`, `IUnitOfWork.cs`), the entity marker interface, the shared `DataAccessException`, and the `Entity<TKey>` base class. `IEntity<TKey>` is the linchpin of that project: every repository query method constrains its `TEntity` to `class, IEntity<TKey>`, which is what lets one repository handle all entity types in the system. If it were removed, those constraints would have to point at something else (for example the `Entity<TKey>` base class directly), coupling the contracts to that implementation detail.

### Relationships

- **`Entity<TKey>`** (`DataAccessLayer/Entity.cs`) — the abstract base class that implements this interface. It provides a `virtual` `Id` property with a protected setter, initialized to `default!` in the constructor. Concrete entities derive from it, e.g. `User : Entity<int>` in `Entity.Hibernate/User.cs`.
- **`IRepository<TKey>`** (`DataAccessLayer/IRepository.cs`) — all seven query methods (`GetQueryable`, `Single`, `SingleOrDefault`, `Find`, `FindOne`, `GetAll`, ...) carry the constraint `where TEntity : class, IEntity<TKey>`.
- **`HibernateRepository<TKey>`** and **`IHibernateRepository<TKey>`** (`DataAccessLayer.Hibernate/`) — the NHibernate-backed implementation, which repeats the same constraint on every method, including `Save<TEntity>`.

The `out TKey` variance marker makes the interface covariant, so `IEntity<int>` is usable wherever `IEntity<System.ValueTypes>`-style broader key types are not needed — in practice it means a reference to an entity with a narrower key can flow through APIs typed on a broader one. Callers of the repository never touch this interface directly; they get it implicitly by their entity classes.

### Behavior & Details

The interface has a single member:

- `TKey Id { get; }` — get-only, so identity is exposed read-only through the contract. The `where TKey : struct` constraint restricts key types to value types (`int`, `Guid`, ...), mirroring the constraint on `Entity<TKey>`.

There is no logic, no events, no other members — it is purely a shape declaration. The only behavioural consequence lives in its consumer `Entity<TKey>`: because `Id` there is `virtual` with a protected setter, a derived class cannot widen the setter through the interface (the interface allows no setter), but it *can* override the property with a public setter, which compiles and bypasses the "identity is not assignable through the contract" intent.

### Rough edges

- The interface declares `Id` read-only, but `Entity<TKey>.Id` is `virtual`, so a derived class could override it with a public setter and silently break the read-only identity guarantee the interface expresses.
- `Entity<TKey>`'s constructor sets `Id = default!`, meaning an unsaved entity has a "valid" default key (e.g. `0` for `int`) rather than a null-like marker — callers checking `Id == default(TKey)` to detect unsaved entities can only work because every entity derives from `Entity<TKey>`, not because the interface guarantees it.