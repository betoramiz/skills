# Testing Strategy

This guide outlines our testing philosophy, folder organization, naming conventions, and the boundaries of unit, integration, and database tests.

---

## 1. Test Organization & Naming Conventions

### File Location
Keep test files located within a central `tests/` directory at the module/feature level. This reduces visual clutter in the source files while keeping the tests close to the implementation.

*   **Domain Entities**: Place inside the `tests/` subfolder (e.g., `src/features/users/tests/User.spec.ts`).
*   **Use Cases**: Place inside the `tests/usecases/` subfolder (e.g., `src/features/users/tests/usecases/create.usecase.spec.ts`).
*   **Hono Routes**: Place inside the `tests/` subfolder (e.g., `src/features/users/tests/users.routes.spec.ts`).
*   **Query Factories**: Place inside the `tests/` subfolder (e.g., `src/features/users/tests/users.queries.spec.ts`).

### File Extensions
Use the `.spec.ts` suffix for all test files. The `vitest` runner automatically discovers these files.

---

## 2. The Testing Pyramid

We classify our tests into four main categories:

```text
   ▲
  / \      DB Integration Tests (Real db connection, slower)
 / I \     Route Integration Tests (app.request in-memory, fast)
/  U  \    Use Case Unit Tests (Mocked query factories, ultra-fast)
/  D  \   Domain Unit Tests (Pure logic, no mocks, ultra-fast)
-------
```

### A. Domain Unit Tests (Pure Business Logic)
*   **Goal**: Validate state validation, domain entity invariants, reconstitution logic, and entity behaviors.
*   **Boundary**: Pure JS/TS functions and classes. No mocks, no databases, no external dependencies.
*   **Speed**: Instantaneous (microsecond execution).

### B. Use Case Unit Tests
*   **Goal**: Validate domain entities orchestration, workflow rules, and Result outcomes (`ok` vs `fail`).
*   **Boundary**: Mock the Query Factory dependencies entirely. Do not import or instantiate database connections (`db`).
*   **Speed**: Ultra-fast (microsecond execution).

### B. Route Integration Tests
*   **Goal**: Verify HTTP transport, JSON parsing, query string parsing, parameter validation (`zValidator`), response headers, and mapping of `Result<T, E>` to HTTP status codes.
*   **Boundary**: Use Hono's `app.request()` method to dispatch in-memory HTTP requests to the app. Mock the use cases or query factories in the container so you don't run real database operations.
*   **Speed**: Fast (milliseconds execution).

### C. Database Integration Tests
*   **Goal**: Verify complex raw queries, Drizzle operators, migrations, constraints (e.g., cascading deletes, unique indices), and transaction commits or rollbacks.
*   **Boundary**: Execute real queries against a dedicated PostgreSQL database instance. Ensure clean state after each test.
*   **Speed**: Slower (hundreds of milliseconds).

---

## 3. Mocking Strategy in Vitest

Vitest provides a robust mocking system via the `vi` object and `vitest` imports:
*   `vi.fn(fn)`: Creates a spy/mock function.
*   `jest`-like matchers: `expect(fn).toHaveBeenCalled()`, `expect(fn).toHaveBeenCalledWith(...)`.

### Mocking Query Factories
Instead of mocking the database client, we mock the **Query Factory returned object**.

```ts
import { vi } from "vitest";
import type { UserQueries } from "../users.queries.ts";

// Create a typed mock factory helper
export const createMockUserQueries = (overrides?: Partial<UserQueries>): UserQueries => {
  return {
    findById: vi.fn(overrides?.findById ?? (async () => null)),
    save: vi.fn(overrides?.save ?? (async () => {})),
    getAll: vi.fn(overrides?.getAll ?? (async () => [])),
    ...overrides,
  };
};
```
This is fully typed! If `UserQueries` changes, the typescript compiler will immediately warn you about missing or misconfigured mock methods.
