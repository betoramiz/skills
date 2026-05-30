# Agent Instructions

Use `.ai/skills/` as the shared project convention source.

For Angular and .NET backend work, read the relevant `SKILL.md` files in `.ai/skills/` before implementing changes. Prefer the most specific skill for the task, starting with the architectural foundations.

### Angular Skills
When working on Angular, the agent MUST read the relevant `SKILL.md` file under `.ai/skills/angular-*` before implementing changes.
- `.ai/skills/angular-architecture/SKILL.md` for feature structure, routes, layers, aliases, and layout conventions.
- `.ai/skills/angular-component/SKILL.md` for standalone components, signal inputs/outputs, template control flow, and component exports.
- `.ai/skills/angular-form-component/SKILL.md` for typed reactive forms, validators, and Material form layouts.
- `.ai/skills/angular-facade/SKILL.md` for feature state, signals, RxJS orchestration, and lifecycle cleanup.
- `.ai/skills/angular-routing/SKILL.md` for app-level layout routes, dashboard feature routes, lazy loading, and redirects.
- `.ai/skills/angular-service/SKILL.md` for REST services, `BaseService`, custom endpoints, and service placement.
- `.ai/skills/angular-material-m3-theming/SKILL.md` for Material 3 theme tokens, Sass mixins, and Tailwind coexistence.
- `.ai/skills/angular-testing/SKILL.md` for component, facade, service, form, and routing tests or regressions.

### .NET Backend Skills
When working on the .NET 10 Clean Architecture backend, the agent MUST read the relevant `SKILL.md` file under `.ai/skills/dotnet-*` before implementing changes.
- `.ai/skills/dotnet-development/SKILL.md` for main .NET backend guidelines (tech stack, build/run commands, and general guidelines).
- `.ai/skills/dotnet-architecture-rules/SKILL.md` for Clean Architecture rules, layer boundary enforcement (Domain, Application, Infrastructure, Api), and rich domain model guidelines.
- `.ai/skills/dotnet-domain/SKILL.md` for domain layer conventions, POCO entities, private setters, factory methods, repository contracts, and ErrorOr integration.
- `.ai/skills/dotnet-application/SKILL.md` for application layer rules, CQRS Use Cases (commands/queries), FluentValidation, `IUseCase` structure, and pagination.
- `.ai/skills/dotnet-api/SKILL.md` for API layer rules, controllers inheriting from `ApiController`, immutable request/response records, Swagger, and endpoint mapping.
- `.ai/skills/dotnet-infrastructure/SKILL.md` for infrastructure layer conventions, EF Core DbContext, Fluent API configurations, migrations, repository implementations, and external integrations (Stripe, Email).
- `.ai/skills/dotnet-data-access/SKILL.md` for CQRS hybrid data access conventions (EF Core for mutations; Dapper for high-performance read-only queries), PostgreSQL driver, and postgres-specific features.
- `.ai/skills/dotnet-testing-guide/SKILL.md` for backend test writing, using xUnit, FluentAssertions, Mocking (Moq/NSubstitute), and mandatory Integration Testing with Testcontainers.

If the environment supports native skills, prefer invoking the matching `$angular-*` or `$dotnet-*` skill name. If not, treat the `SKILL.md` files above as required project context.
