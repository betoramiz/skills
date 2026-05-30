---
name: dotnet-domain
description: Conventions for Domain Layer in FloreriaStore project.
---

## Domain Layer Rules

### Purpose
The domain layer is the heart of the application. It contains the business logic and entities that define the business domain. It has ZERO external dependencies.

### Project Structure
- **Floreria.Domain**: Contains enterprise logic, entities, and repository interfaces. Also contains DTO classes that work as the response from the Repository Operations. No external dependencies allowed.

### Key Principles
- **No External Dependencies**: This layer must not reference any other project (Infrastructure, Application, or external libraries).
- **Business Logic Only**: Contains only business rules and entities.
- **No Persistence Logic**: Do NOT include Entity Framework Core, Dapper, or any database-related code here.
- **DTOs**: DTOs are allowed in this layer only to be used as the response from the Repository Operations.
- **No Validation Logic**: Validation logic belongs in the Application layer (FluentValidation).

### Code Patterns
Detailed code examples and structured patterns are organized inside the `references/` folder:
- [Rich Domain Entity Pattern](references/rich-entity.md): State encapsulation, factory methods, and business invariants using ErrorOr.
- [Repository Interface Pattern](references/repository-interface.md): Abstraction contracts and dedicated request/response DTOs.
- [Domain Unit Test Pattern](references/domain-unit-test.md): Isolated unit testing of enterprise business logic and rules.

### Entities
- Entities should be POCOs (Plain Old CLR Objects).
- Use private setters for internal state management.
- Expose public methods for business operations.
- Each entity is located in a folder with the same name as the entity.
- This layer uses the `ErrorOr` library to return the result of the operations.

### Interfaces
- Define interfaces for repository abstractions (if needed).
- We do not use generic repository pattern.
- We use DTOs to return the result of the operations.
- We use Repository for operations that involve fetching data from the database and require complex queries.

### Validation
- Validation should be done in the Application layer (FluentValidation).
- Domain entities should only enforce invariants through private setters and factory methods.

### Testing
- Unit tests should focus on business logic and invariants.
- No database dependencies in domain tests.

### Common Mistakes to Avoid
- ❌ Including EF Core DbContext in domain layer
- ❌ Exposing public setters for all properties
- ❌ Including validation logic here
- ❌ Creating unnecessary abstractions

### When to Modify Domain Layer
- When business rules change
- When new entities are needed
- When invariants need to be enforced
- When domain events are required

### When NOT to Modify Domain Layer
- When adding new API endpoints
- When creating new queries or commands
- When implementing database access
- When adding validation logic
