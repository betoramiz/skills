---
name: ts-architecture
description: Architecture guide for the web-api TypeScript backend. Use when Codex needs to add or refactor backend features, modules, routes, database schemas, query factories, containers, shared infrastructure, Hono HTTP wiring, Drizzle/Postgres persistence, or cross-layer conventions in this Bun + Hono + Drizzle API.
---

# TS Architecture

## Overview

Use this skill to preserve the architecture of `web-api` while changing or extending the backend. The project is a Bun TypeScript API organized around feature modules, Hono routes, application use cases, domain entities, Drizzle query factories, and shared response/error utilities.

## Workflow

1. Inspect the target feature under `src/features/<feature>/` before editing.
2. Keep feature code grouped by module:
   - `<feature>.routes.ts` for Hono adapters.
   - `<feature>.schemas.ts` for Zod request validation and command types.
   - `<feature>.dtos.ts` for HTTP response interfaces (what the API returns to clients).
   - `<feature>.queries.ts` for Drizzle persistence operations.
   - `<feature>.container.ts` for dependency wiring.
   - `<Entity>.ts` for domain creation, reconstitution, and primitives.
   - `usecases/*.usecase.ts` for application operations.
   - `usecases/errors.ts` for expected application/domain errors.
3. Keep shared infrastructure in `src/shared/` and database infrastructure in `src/db/`.
4. Wire new feature routes from `src/index.ts` with `app.route('/resource', routes)`.
5. Prefer path aliases from `tsconfig.json`: `@db/*` and `@shared/*`.
6. Preserve explicit `.ts` import extensions used throughout the codebase.
7. Validate expected failures with `Result<T, ErrorResponse>` and `AppError`; do not throw for normal application conditions such as not found or domain validation failures.
8. Let unexpected database errors bubble to `app.onError`, where `parseDbError` maps known Postgres errors.

## Layer Responsibilities

- HTTP routes adapt Hono requests to use cases, run `zValidator`, and call `respond` for `Result` values.
- Zod schemas define external input shape and export inferred command types.
- DTOs (`<feature>.dtos.ts`) define the response interfaces returned to clients. Use cases import response types from here — never define response interfaces inline inside use case files.
- Use cases orchestrate domain objects and query factories; they should not know about Hono contexts.
- Domain entities enforce invariants through static factories returning `Result<Entity, ErrorResponse>`. The entity class is its own type — do not define a separate `Props` interface alongside it.
- Query factories receive `Db`, perform Drizzle operations, and map rows to DTO projections or accept domain entity instances for persistence.
- Containers create query factories and use case instances, then pass them to route factories.
- Shared helpers provide result, response, validation, and error handling primitives.

## Type Boundaries

- **Command types** (`<feature>.schemas.ts`): inferred from Zod — describe what comes *in* via HTTP.
- **DTO types** (`<feature>.dtos.ts`): plain interfaces — describe what goes *out* via HTTP. Use cases return these wrapped in `Result`.
- **Domain entity** (`<Entity>.ts`): instantiated class — encapsulates invariants. Query factories accept entity instances; they map to `NewX` Drizzle insert types internally via a private `toInsert` helper.
- **Drizzle schema types** (`src/db/schemas/`): `$inferSelect` / `$inferInsert` — used only inside query factories, never leaked to use cases or routes.

## References

- Read `references/project-map.md` when you need a concise map of the current architecture and file responsibilities.
- Read `references/feature-module-example.md` before adding a new feature module or expanding an existing one with routes, persistence, schemas, and use cases.

## Checks

Before finishing an architecture change:

- Ensure every new database table is exported from `src/db/schemas/index.ts`.
- Ensure every new route is wired into `src/index.ts`.
- Ensure `zValidator` uses `validationErrorAsJson`.
- Ensure route handlers do not duplicate `respond` logic for `Result` values.
- Ensure query factories accept `Db` instead of importing `db` directly.
- Ensure expected errors return `fail(AppError.*(...))`.
- Run the available TypeScript or Bun validation command if the project provides one.
