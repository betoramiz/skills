---
name: dotnet-development
description: Specialized rules and instructions for .NET 10 backend development in the FloreriaStore project.
---

# FloreriaStore Backend Rules

This guide provides specific instructions for modifying the .NET 10 backend of this project.

## Tech Stack
- .NET 10, ASP.NET Core
- Entity Framework Core with PostgreSQL
- FluentValidation for request validation
- Swagger for API documentation

## Project Structure
We use Clean Architecture. When adding new features, follow this structure:
- **Floreria.Domain**: Contains enterprise logic, entities, and domain interfaces. No external dependencies allowed.
- **Floreria.Application**: Contains business logic, commands, queries, and DTOs. Depends only on `Domain`.
- **Floreria.Infrastructure**: Contains external implementations like Entity Framework Core DB Context, third-party services, etc.
- **Floreria.Api**: The entry point (Controllers, Program.cs). Depends on `Application` and `Infrastructure`.

## Building and Running
- To build the backend: `dotnet build Floreria.sln`
- To run tests: `dotnet test Floreria.sln`
- The API is typically run via Docker as part of the total project.

## Guidelines
- **Dependency Injection**: Use DI everywhere. Do not use static state or `new` keyword for services.
- **CQRS**: Prefer command/query separation for read and write operations when appropriate.
- **Database Changes**: Modifying entities in `Floreria.Domain` usually implies updating or creating an EF Core migration in the infrastructure project.

## Specific Topics
For detailed guidelines on specific areas, the agent MUST read the following files according to the task:
- When modifying architectural layers or creating new ones: `read dotnet-architecture-rules/SKILL.md`
- When working with the Database, EF Core, Dapper, or Postgres: `read dotnet-data-access/SKILL.md`
- When creating new Use Cases, Commands, Queries, or Validations: `read dotnet-application/SKILL.md`
- When creating API Endpoints or handling HTTP responses: `read dotnet-api/SKILL.md`
- When writing tests (Unit or Integration): `read dotnet-testing-guide/SKILL.md`
