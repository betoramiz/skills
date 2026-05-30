---
name: dotnet-data-access
description: Conventions for using Entity Framework Core, Dapper, and PostgreSQL in the FloreriaStore project.
---

# Data Access Conventions

The backend uses a hybrid data access approach using PostgreSQL as the database.

## CQRS Data Access Golden Rule
- **Commands (Mutations - INSERT/UPDATE/DELETE):** Use **Entity Framework Core**. It excels at change tracking, complex domain mutations, and maintaining consistency.
- **Queries (Reads - SELECT):** Use **Dapper**. It provides maximum performance and flexibility for reading flat data, generating reports, and view models.

## Entity Framework Core
- **Configuration:** Use the `IEntityTypeConfiguration<T>` interface (Fluent API) to configure your entities in `Floreria.Infrastructure`.
- **Forbidden:** Do NOT use Data Annotations (like `[Table("Name")]` or `[Required]`) in `Floreria.Domain` entities. Keep the domain clean of EF Core details.
- **Migrations:** Migrations must reside in the Infrastructure project. When making domain changes that affect the database schema, generate a new migration.

## Dapper
- **Queries:** Keep SQL Queries organized. Avoid inline SQL strings scattered in logic. Use constants, static resource files, or embedded `.sql` scripts.
- **Mapping:** Map query results directly to simple DTOs defined in `Floreria.Application`. Don't map Dapper results back to Domain Entities unless absolutely necessary.

## PostgreSQL
- **Driver:** Use `Npgsql`.
- **Naming Conventions:** Follow Postgres standard naming (snake_case) at the database level where appropriate, usually mapped automatically by EF Core naming conventions.
- **Data Types:** Make use of PostgreSQL specific features when appropriate, such as `jsonb` for dynamic data and UUIDs for primary keys where applicable.
