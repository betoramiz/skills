---
name: dotnet-infrastructure
description: Conventions for Infrastructure Layer in FloreriaStore project.
---

## Infrastructure Layer Rules

### Purpose
The infrastructure layer is responsible for implementing the interfaces defined in the application layer. It contains the database access code, external service integrations, and other infrastructure concerns.

### Project Structure
- **Floreria.Infrastructure**: Contains the implementation of the interfaces defined in the application layer.

### Key Principles
- **Implements Application Layer Interfaces**: This layer must implement the interfaces defined in the application layer.
- **Contains Database Access Code**: This layer contains the Entity Framework Core DbContext and repository implementations.
- **Contains External Service Integrations**: This layer contains the implementation of external service integrations.

### Entity Framework Core
- **DbContext**: The DbContext should be defined in this layer.
- **Migrations**: Migrations should be defined in this layer.
- **Configuration**: Entity configurations should be defined in this layer using the `IEntityTypeConfiguration<T>` interface.

### Repository Implementations
- Repository implementations should be defined in this layer.
- Repository implementations should implement the interfaces defined in the application layer.

### External Service Integrations
- External service integrations should be defined in this layer.
- External service integrations should implement the interfaces defined in the application layer.

### Common Mistakes to Avoid
- ❌ Including EF Core DbContext in application layer
- ❌ Exposing public setters for all properties
- ❌ Including validation logic here
- ❌ Creating unnecessary abstractions

### When to Modify Infrastructure Layer
- When database access needs to be implemented
- When external service integrations need to be implemented
- When EF Core DbContext needs to be modified
- When migrations need to be created

### When NOT to Modify Infrastructure Layer
- When business logic needs to be modified
- When new commands or queries are needed
- When validation logic needs to be added
- When DTOs need to be created
