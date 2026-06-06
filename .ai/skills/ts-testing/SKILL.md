---
name: ts-testing
description: Strategy, conventions, and patterns for writing unit tests, route integration tests, and database integration tests in this Bun + Hono + Drizzle API using Vitest.
---

# Testing Strategy & Guia de Pruebas

## Overview

Use this skill when writing tests for any part of the application, including Hono HTTP routes, Clean Architecture Use Cases (`usecases/`), Domain entities, and Drizzle query factories. All tests use the fast and powerful `Vitest` testing framework.

## Workflow

1. **Domain Logic Verification**: Write unit tests for Domain Entities (e.g., `src/features/<feature>/<Entity>.ts`) to verify state validation, invariant checks, and reconstitutions in absolute isolation without any mocks.
2. **Establish Mock boundaries for Use Cases**: Write unit tests for Use Cases by mocking their dependent query factories. Use Cases should have no DB connections.
3. **Hono Route Integration**: Test HTTP routing, middleware validation (`zValidator`), and output formatting by calling `app.request()` in memory.
4. **Database Integration (Optional/Advanced)**: Test custom or complex SQL query factories against a live test Postgres database.
5. **Run Tests**: Execute the test runner locally using `bun run test` (which triggers `vitest run`) or specify a target pattern (e.g., `bun run test src/features/users/`).

## References

- Read `references/testing-strategy.md` for guidance on test design principles, mocking, handling asynchronous tests, and organizing tests by folder and extension.
- Read `references/test-examples.md` for complete, production-grade test implementations:
  1. Use Case unit test with typed mock query factories.
  2. Route integration test utilizing Hono's in-memory `app.request` client.
  3. Database integration test with query verification.

## Checks

Before finishing a feature or refactor:

- [ ] Verify you have written unit tests for any new or modified Domain Entities.
- [ ] Verify you have written unit tests for any new or modified Use Cases.
- [ ] Verify you have tested edge cases (e.g. invalid inputs, not found errors, validation constraints).
- [ ] Confirm Hono routes are tested for correct HTTP Status Codes (e.g., 200, 201, 400, 404, 409) and error schemas.
- [ ] Ensure tests do not leak open handles (e.g., close DB connections or mock external network calls).
- [ ] Run `bun run test` and ensure all tests pass successfully.
