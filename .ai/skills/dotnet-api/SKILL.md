---
name: dotnet-api
description: Conventions for API Layer in FloreriaStore project.
---

## API Layer Rules

### Purpose
The API layer is responsible for exposing the application's functionality to the outside world. It contains the API endpoints, controllers, and request/response models.

### Project Structure
- **Floreria.Api**: Contains the API endpoints, controllers, and request/response models.

### API Controller
- API endpoints use Controllers to expose the application's functionality.
- Each controller inherits from `ApiController` class.
- Each controller implements the UseCase Repository defined in the Application Layer.

For standard code patterns, refer to:
- [API Controller Pattern](references/api-controller.md)
- [Pagination Response Pattern](references/pagination-response.md)

- Controllers that call a use case that does not return an ErrorOr should handle it appropriately.
- API endpoints should be defined in this layer using the `IEndpointRouteBuilder` interface.
- API endpoints should use the `IUseCase` interface to execute the application's functionality.
- API endpoints should return `ErrorOr<T>` where T is the response DTO or some Struct coming from the ErrorOr library.

### Request/Response Models
- Request and response models should be defined in this layer using the `record` keyword.
- Request and response models should be immutable.
- Request and response models should be validated using FluentValidation.

### Common Mistakes to Avoid
- ❌ Including EF Core DbContext in application layer
- ❌ Exposing public setters for all properties
- ❌ Including validation logic here
- ❌ Creating unnecessary abstractions

### When to Modify API Layer
- When new API endpoints are needed
- When request/response models need to be created
- When API endpoints need to be modified
- When request/response models need to be modified

### When NOT to Modify API Layer
- When business logic needs to be modified
- When database access needs to be implemented
- When domain logic needs to be modified
- When validation logic needs to be added
