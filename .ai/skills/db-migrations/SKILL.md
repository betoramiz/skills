---
name: db-migrations
description: Database schema definitions, Drizzle migrations, seeding, and advanced query patterns (including joins, relationships, and transactional use cases) in this Bun + Hono + Drizzle API.
---

# Database Migrations & Advanced Drizzle ORM

## Overview

Use this skill when defining database tables, generating or executing migrations, writing seed files, or implementing advanced Drizzle ORM queries (such as transactions, complex joins, and subqueries) inside the `web-api` project.

## Workflow

1. **Schema Definition**: Define or modify tables in `src/db/schemas/<name>.schema.ts`.
2. **Export Registration**: Ensure all tables are explicitly exported in `src/db/schemas/index.ts`.
3. **Migration Generation**: Run `bun run db:generate:migration` to produce the SQL migration files under `src/drizzle/`.
4. **Migration Application**: Apply migrations locally using `bun run db:migrate`.
5. **Query Integration**: Inject the `Db` instance into feature query factories (`src/features/<feature>/<feature>.queries.ts`) to query the newly created schema.
6. **Transactional Logic**: When an operation spans multiple tables or writes that must be atomic, implement Drizzle's transactional API in the use case or query factory.

## References

- Read `references/migrations-guide.md` for a comprehensive step-by-step guide on creating tables, generating and executing migrations, resolving configuration mismatches, and seeding data.
- Read `references/drizzle-patterns.md` for developer-proven patterns regarding complex Drizzle queries, relational joins, subqueries, and clean architecture-compliant transaction handling.

## Checks

Before finishing a database or query change:

- [ ] Confirm the new table schema is defined under `src/db/schemas/` with matching camelCase TS properties and snake_case database columns.
- [ ] Confirm the table is exported in `src/db/schemas/index.ts`.
- [ ] Confirm you ran `bun run db:generate:migration` and checked the generated `.sql` files inside `src/drizzle/` for correctness.
- [ ] Confirm you ran `bun run db:migrate` successfully.
- [ ] Confirm no direct `db` database connections are imported in your queries file; it must accept a generic `Db` instance.
- [ ] Verify that transactions are executed using the transaction context client (`tx`) rather than the global `db` connection.
