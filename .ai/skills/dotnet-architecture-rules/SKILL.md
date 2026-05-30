---
name: dotnet-architecture-rules
description: Strict Clean Architecture rules for the FloreriaStore .NET 10 Backend.
---

# Clean Architecture Rules

This project strictly adheres to Clean Architecture principles. Do not violate the dependency rule: dependencies must point inward toward the Domain.

## 1. Floreria.Domain
The core of the system.
- **Allowed:** Entities, Value Objects, Domain Events, Domain Exceptions, Repository Interfaces.
- **Forbidden:** External dependencies (Nuget packages like EF Core, Dapper), UI concerns, database details.
- **Models:** Use Rich Domain Models. Behavior and data should be encapsulated. Avoid Anemic Domain Models (models with only public getters and setters and no logic).

## 2. Floreria.Application
Contains the application's business logic, Use Cases, CQRS Commands/Queries.
- **Allowed:** DTOs, CQRS Handlers (Commands/Queries), FluentValidation rules.
- **Dependencies:** Depends ONLY on `Floreria.Domain`.
- **Forbidden:** Direct database access, HTTP contexts, UI concerns.

## 3. Floreria.Infrastructure
External concerns and implementations.
- **Allowed:** Entity Framework Core `DbContext`, Dapper implementations, Repositories, Migrations, 3rd party API clients (Email, Stripe, etc.).
- **Dependencies:** Depends on `Floreria.Application` and `Floreria.Domain`.

## 4. Floreria.Api (Presentation)
The entry point of the application.
- **Allowed:** Controllers, Minimal APIs, Middlewares, Dependency Injection configuration (`Program.cs`).
- **Dependencies:** Depends on `Floreria.Application` (for routing requests) and `Floreria.Infrastructure` (strictly for Dependency Injection setup only).
- **Forbidden:** Injecting `DbContext` directly into Controllers. Use the injected Application Services.
