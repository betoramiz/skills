---
name: dotnet-application
description: Conventions for Application Layer in FloreriaStore project.
---

# Application Layer Rules

## Purpose
The application layer is the orchestrator of the application. It contains the business logic, commands, queries, and DTOs.

## Project Structure
- **Floreria.Application**: Contains business logic, commands, queries, and DTOs. Depends only on `Domain`.

## Key Principles
- **CQRS**: Prefer command/query separation for read and write operations when appropriate.
- **DTOs**: DTOs are allowed in this layer only to be used as the response from the Repository Operations and the response from UseCases.
- **Validation Logic**: Validation logic belongs in the Application layer (FluentValidation).
- **UseCases**: UseCases are used to orchestrate the application. Each UseCase is located in a folder with the same name under the `Features` folder.

## Internal Structure
- `Common`: Contains shared functionality and cross-cutting concerns (e.g. authorization, pagination, custom errors, common interfaces).
- `Data`: Contains data access contracts and contracts, primarily the `IBackendDbContext` interface.
- `EcommerceFeatures`: Encapsulates e-commerce storefront API UseCases.
- `Features`: Contains administrative platform backend UseCases (e.g. Category, Product management).
- `Helpers`: Dedicated to shared utility methods and standalone helper classes.

## Code Patterns
Detailed code examples and structured patterns are organized inside the `references/` folder:
- [Validation Pattern](references/validator.md): Implementing validators with FluentValidation.
- [Command Use Case Pattern](references/command-use-case.md): Structural layout of state-mutating commands.
- [Query Use Case Pattern](references/query-use-case.md): Structural layout of queries and paginated lists.
- [Use Case Repository Pattern](references/use-case-repository.md): Feature orchestration registry.

## Use Cases Guidelines
- Each Use Case should be named with the action that it performs.
- Each Use Case sould implement the markup interface `IUseCase`.
- Each Use Case must implement the main method `ExecuteAsync`.
- Use Cases that perform a Command contain a Validator class implementing `AbstractValidator<Command>`.
- The `ExecuteAsync` method must validate commands using the validation helper helper class before executing.
- Paginated responses return `PaginationResponse<T>` instead of `ErrorOr<T>`.
