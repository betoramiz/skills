---
name: ts-usecases
description: Use case implementation guide for the web-api TypeScript backend. Use when Codex needs to create, modify, review, or debug application use cases under src/features/*/usecases, including Result-based success/failure flows, command and response DTOs, domain entity orchestration, query dependency injection, and expected application errors.
---

# TS Use Cases

## Overview

Use this skill when implementing application behavior inside `src/features/<feature>/usecases/`. Use cases are small classes with injected query dependencies, an `execute` method, and explicit `Result` flows for expected failures.

## Workflow

1. Inspect the feature entity, schemas, queries, routes, and existing use cases before editing.
2. Define the input command in `<feature>.schemas.ts` when it comes from HTTP input.
3. Define the output interface in `<feature>.dtos.ts` and import it from there. Never declare response interfaces inline inside a use case file.
4. Inject query factories through the constructor using the feature query type, such as `UserQueries`.
5. Keep Hono, route contexts, and HTTP response formatting out of use cases.
6. Return `ok(value)` for successful operations that use `Result`.
7. Return `fail(AppError.*(...))` or a helper from `usecases/errors.ts` for expected failures.
8. Let unexpected persistence errors throw so the global Hono error handler can parse known database errors.

## Patterns

- Create operations call the entity static factory (`Entity.create(...)`), check `result.ok`, persist via queries, and return a DTO (e.g. `{ idCreated }`).
- Read-by-id operations query by identifier, return a feature-specific not-found helper when missing, and map domain objects to response DTOs.
- List operations may return direct projections if the route is intentionally direct; prefer `Result` when the route uses `respond`.
- Domain validation belongs in the entity when it protects invariants.
- Request shape validation belongs in Zod schemas and route validators.

## Domain Entity Convention

Entities use a **private constructor + static factory** pattern. The class itself is the type — no separate `Props` interface:

```ts
export class Card {
  private constructor(
    public readonly name: string,
    public readonly bank: string,
    public readonly billingCutoffDay: number,
    public readonly lastPaymentDate: string,
  ) {}

  static create(name: string, bank: string, billingCutoffDay: number, lastPaymentDate: string): Result<Card, ErrorResponse> {
    if (!name.length) return fail(AppError.BadRequest('Nombre es requerido'));
    // ... other validations
    return ok(new Card(name, bank, billingCutoffDay, lastPaymentDate));
  }
}
```

Use cases receive `card.value` (a `Card` instance) and pass it directly to query factories.

## References

- Read `references/usecase-patterns.md` when choosing the correct use case shape for create, get, list, update, delete, or domain operations.
- Read `references/detailed-examples.md` for complete examples that match this project's imports, `Result` handling, query injection, and error helpers.
- Read `references/auth-examples.md` for real authentication use cases showing bcrypt password hashing, JWT token generation, and auth middleware patterns.

## Checks

Before finishing a use case change:

- Confirm the route passes a validated command or primitive values into `execute`.
- Confirm the use case only depends on query interfaces and domain/shared primitives.
- Confirm all expected failures are represented with `Result` instead of thrown exceptions.
- Confirm response DTOs do not expose fields the route should hide.
- Confirm any new query method is implemented in `<feature>.queries.ts` and included in the returned object type.
- Run the available TypeScript or Bun validation command if the project provides one.
