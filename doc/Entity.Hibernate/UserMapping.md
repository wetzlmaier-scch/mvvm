# `Entity.Hibernate/UserMapping.cs`

### Purpose

`UserMapping` is the Fluent NHibernate mapping for the `User` entity in the `Entity.Hibernate` project. It tells NHibernate how the in-memory `User` class (`Entity.Hibernate/User.cs`) translates to a database table — table name, primary key strategy, and column constraints — so the ORM can generate SQL for it.

### Role in the Codebase

This class is one of two files in the `Entity.Hibernate` project: `User.cs` defines the entity and this file defines its persistence mapping. It contains no application logic of its own; it is consumed indirectly by NHibernate at session-factory build time.

The session factory is built by `SessionFactoryBuilder.BuildSessionFactory()` in `DataAccessLayer.Hibernate/SessionFactoryBuilder.cs`, which loads every mapping assembly passed to it and calls `FluentMappings.AddFromAssembly(...)` on each. That call discovers all `ClassMap<T>` subclasses in the assembly — including this class — via reflection, so the mapping is picked up purely by being present in the assembly. The `BusinessLayer` project references `Entity.Hibernate`, which is why this mapping ends up in scope for the application's data access.

If this file were removed, `User` would have no mapping in the assembly: NHibernate would not know the `Users` table exists, and any repository operation on `User` would fail at query/insert time.

### Relationships

- **`User`** (`Entity.Hibernate/User.cs`) — the entity this class maps. It extends `DataAccessLayer`'s abstract `Entity<int>` base class (defined in `DataAccessLayer/Entity.cs`), which provides the `Id` property. The generic parameter of `ClassMap<User>` is what links the two files; there is no direct code dependency, only the convention that `ClassMap<T>` must reference an actual entity class.
- **FluentNHibernate** (external library) — `ClassMap<T>` comes from `FluentNHibernate.Mapping`.
- **Callers** — none call this class directly. `SessionFactoryBuilder` (and similarly `HibernateRepositoryFactory` in the same project) discover it by scanning the assembly for `ClassMap<T>` subclasses.

### Behavior & Details

All mapping configuration happens in the constructor:

- `Table("Users")` — the entity maps to a table named `Users`, overriding the convention-based name.
- `Id(x => x.Id).GeneratedBy.Identity()` — the `Id` property (inherited from `Entity<int>`) maps to an identity column; the database assigns the value on insert, so the application never sets it.
- `Map(x => x.FirstName).Not.Nullable().Length(50)` and the same for `LastName` — both string columns are `NOT NULL` with a 50-character limit.
- `Map(x => x.IsActive).Not.Nullable()` — the `bool` column is `NOT NULL` with no length constraint.

Two caveats worth knowing when changing this mapping:

1. Because `SessionFactoryBuilder` runs `SchemaUpdate` against the configuration, NHibernate will auto-update the database schema to match this mapping when the factory is built. Change a property here and the schema change follows (additively, per how `SchemaUpdate` works).
2. Changing a property name on `User` without updating this file breaks the mapping at factory-build time, since the `x => x.Property` expressions are compiled against the entity's actual members.