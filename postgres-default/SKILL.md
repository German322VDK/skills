---
name: postgres-default
description: PostgreSQL standard for corporate .NET and EF Core projects. Use when creating, configuring, reviewing, or refactoring PostgreSQL database setup, EF Core PostgreSQL mappings, Npgsql configuration, migrations, SQL scripts, indexes, schemas, naming conventions, or local development database configuration.
---

# Postgres Default

## Purpose

Use this skill when creating, configuring, reviewing, or refactoring PostgreSQL database setup, EF Core PostgreSQL mappings, migrations, SQL scripts, indexes, schemas, naming conventions, or local development database configuration.

Prefer consistency with the existing project. If an existing project already has a clear PostgreSQL convention, extend it instead of introducing a parallel style.

## Local Development Defaults

Default local PostgreSQL connection:

- host: `localhost`;
- port: `5432`;
- username: `dev`;
- password: `dev`.

Default database name:

- use the project name;
- prefer lowercase `snake_case`;
- example: `TaskTracker` -> `task_tracker`.

Default schema: `public`.

Rules:

- do not create extra schemas unless explicitly requested;
- do not use random database names;
- do not use production-like credentials for local development;
- do not hardcode local credentials into production configuration.

## Database Creation

When creating a local development database, use the project name as the database name.

```sql
CREATE DATABASE task_tracker;
```

Rules:

- use lowercase `snake_case`;
- create only the requested database;
- keep local setup simple and reproducible;
- avoid production-like credentials in local setup instructions.

## Naming Convention

Use `snake_case` for PostgreSQL objects.

Tables:

```text
table_number1
task_items
postamat_documents
analytics_events_outbox
```

Columns:

```text
id
task_id
created_at_utc
update_date
document_type
is_deleted
```

Indexes:

```text
ix_task_items_created_at_utc
ix_postamat_documents_update_date
```

Primary keys:

```text
pk_task_items
```

Foreign keys:

```text
fk_order_items_orders_order_id
```

Unique constraints:

```text
ux_users_email
```

Rules:

- no PascalCase table names;
- no camelCase column names;
- no quoted identifiers by default;
- avoid spaces, Russian names, and mixed casing;
- prefer plural table names;
- keep names short but meaningful.

## EF Core And Npgsql

Use the official Npgsql EF Core provider for PostgreSQL.

Rules:

- keep `DbContext` and EF Core mappings in Infrastructure;
- configure table, column, index, key, and constraint names explicitly when needed to preserve `snake_case`;
- do not rely on quoted PascalCase identifiers;
- keep EF Core attributes out of domain entities when Fluent API can express the mapping;
- use separate `IEntityTypeConfiguration<TEntity>` files for entity configuration;
- use PostgreSQL-specific APIs only in Infrastructure;
- use UTC timestamps for persisted time values.

Preferred shape:

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
{
    options.UseNpgsql(connectionString);
});
```

If the project uses a centralized `AppConfig` / `IOptions<AppConfig>` configuration pattern, read the connection string through that existing pattern.

## Migrations

Keep EF Core migrations deterministic and reviewable.

Rules:

- create migrations from the Infrastructure project unless the solution has another established migration project;
- inspect generated migrations before accepting them;
- do not leave accidental table renames caused only by casing or naming convention changes;
- avoid mixing unrelated schema changes in one migration;
- do not manually edit model snapshot unless fixing a known EF Core metadata issue;
- include indexes, constraints, and column nullability intentionally.

## Indexes And Constraints

Create indexes for real query patterns, not by habit.

Rules:

- index foreign keys when they are used for joins or filtering;
- add unique constraints for real business uniqueness, such as `ux_users_email`;
- name indexes and constraints explicitly when the generated name is unclear or violates the naming convention;
- avoid redundant indexes that duplicate primary keys or unique constraints;
- consider partial indexes for filtered PostgreSQL queries when useful;
- avoid adding indexes without understanding write overhead.

## SQL Scripts

SQL scripts should be safe, readable, and repeatable.

Rules:

- use lowercase `snake_case` object names;
- avoid quoted identifiers by default;
- avoid destructive SQL in setup scripts unless explicitly requested;
- keep local dev scripts separate from production migration scripts;
- do not embed production secrets in SQL files;
- prefer EF Core migrations for application schema evolution unless the project explicitly manages schema through SQL scripts.

## Schemas

Use `public` by default.

Rules:

- do not create extra schemas unless explicitly requested or already established by the project;
- if schemas are required, use lowercase `snake_case` names;
- keep schema ownership clear;
- do not split schemas only for cosmetic organization.

## Connection Strings

Connection strings belong to application configuration, not code.

Rules:

- keep local connection strings in local appsettings, user secrets, environment variables, or dev compose files;
- do not hardcode production connection strings;
- validate required database configuration on startup;
- fail fast when required connection settings are missing;
- do not log passwords or full connection strings.

Local development example:

```text
Host=localhost;Port=5432;Database=task_tracker;Username=dev;Password=dev
```

## Final Validation

Before completing a PostgreSQL task, verify:

- local database name is based on the project name and uses lowercase `snake_case`;
- default local connection uses `localhost:5432`, username `dev`, and password `dev` unless the project already uses another convention;
- default schema remains `public` unless extra schemas were explicitly requested;
- tables, columns, indexes, keys, and constraints use `snake_case`;
- no quoted identifiers were introduced by default;
- EF Core PostgreSQL configuration stays in Infrastructure;
- migrations were inspected and do not contain accidental renames;
- indexes and constraints match real query and business requirements;
- connection strings come from configuration and required values fail fast when missing.
