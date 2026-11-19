# `Entity.Hibernate/UserMapping.cs`

### Purpose

`UserMapping` is the Fluent NHibernate mapping for the `User` entity in the `Entity.Hibernate` project. It declares how the in-memory `User` class translates to the `Users` table: which properties become columns, which are required, and how the primary key is generated. Without it, NHibernate would have no way to persist or load `User` objects.

### Role in the Codebase

The `Entity.Hibernate` project is a small pair of files — the `User` entity and this mapping — whose only job is to describe the `User` domain object's persistence shape. This file is the mapping half of that pair. It is discovered at startup, not referenced directly: `SessionFactoryBuilder.BuildSessionFactory` in `DataAccessLayer.Hibernate/SessionFactoryBuilder.cs` calls `FluentMappings.AddFromAssembly` over the list of mapping assembly names, and Fluent NHibernate finds `UserMapping` there via reflection. Removing it would leave `User` unmapped, so any query or save touching `User` would fail once the session factory is built.

### Relationships

- **`User`** (`Entity.Hibernate/User.cs`): the entity this class maps. Each `Map` call in the constructor targets one of its public properties, so the two files must stay in sync — renaming a property on `User` breaks this mapping at compile time.
- **`ClassMap<T>`**: from the external FluentNHibernate library (referenced in `Entity.Hibernate.csproj`). `UserMapping` inherits the mapping-builder API (`Table`, `Id`, `Map`) from it.
- **`SessionFactoryBuilder`** (`DataAccessLayer.Hibernate/SessionFactoryBuilder.cs`): the indirect consumer. It loads this assembly's Fluent mappings and additionally runs a `SchemaUpdate`, so the column definitions declared here (`not null`, length 50, identity key) are what gets created/updated in the SQL Server database.

### Behavior & Details

All configuration happens in the constructor, since Fluent NHibernate instantiates the mapping and reads its state when building the session factory:

- `Table("Users")` names the database table; NHibernate's default would already be `Users` from the entity name, but this pins it explicitly so a rename of the class does not silently change the table name.
- `Id(x => x.Id).GeneratedBy.Identity()` maps the `Id` property inherited from `Entity<int>` (defined in `DataAccessLayer/Entity.cs`, with a protected setter) to an identity/`auto-increment` column. The database assigns the key on insert; the application never sets it.
- `FirstName` and `LastName` map to `NOT NULL` columns with length 50.
- `IsActive` maps to a `NOT NULL` boolean column. Note the column constraint, not the value: a new `User` built through its public constructor has `IsActive = true` by virtue of `User.cs`, and the mapping only guarantees the column cannot store `NULL`.

There is no logic beyond these declarations — no overrides, no lazy-loading hints, no explicit column names (NHibernate defaults the column names to the property names).

### Rough edges

- The schema is shaped by `SchemaUpdate` executing in `SessionFactoryBuilder` against the mappings' column declarations, so the "50 character / not null" rules live here rather than in a migration. Changing them changes the live database on next startup; there is no separate migration script in this project guarding those columns.