# Production-Grade Testing Examples

This guide contains complete, compile-ready examples of Unit, Route Integration, and Database Integration tests utilizing `Vitest`.

---

## 1. Use Case Unit Test (`src/features/users/tests/usecases/create.usecase.spec.ts`)

In this test, we verify the business logic of `CreateUser` in complete isolation from HTTP routes and PostgreSQL.

```ts
import { describe, it, expect, vi, beforeEach } from "vitest";
import { CreateUser } from "../../usecases/create.usecase.ts";
import type { UserQueries } from "../../users.queries.ts";

describe("CreateUser UseCase", () => {
  let mockUserQueries: UserQueries;
  let useCase: CreateUser;

  beforeEach(() => {
    mockUserQueries = {
      findById: vi.fn(async () => null),
      save: vi.fn(async () => {}),
      getAll: vi.fn(async () => []),
    };

    useCase = new CreateUser(mockUserQueries);
  });

  it("debe fallar si el nombre del usuario tiene menos de 3 caracteres", async () => {
    const command = {
      name: "Ab", // Nombre muy corto (debe ser >= 3)
      email: "test@example.com",
    };

    const result = await useCase.execute(command);

    expect(result.ok).toBe(false);
    if (!result.ok) {
      expect(result.error.statusCode).toBe(400);
      expect(result.error.message).toContain("El nombre debe tener al menos 3 caracteres.");
    }

    // Verificar que no se llamó a la persistencia
    expect(mockUserQueries.save).not.toHaveBeenCalled();
  });

  it("debe crear un usuario exitosamente si el nombre y email son válidos", async () => {
    const command = {
      name: "John Doe",
      email: "john@example.com",
    };

    const result = await useCase.execute(command);

    expect(result.ok).toBe(true);
    if (result.ok) {
      expect(result.value.name).toBe("John Doe");
      expect(result.value.email).toBe("john@example.com");
      expect(result.value.id).toBeDefined();
      expect(result.value.isActive).toBe(true);
    }

    // Verificar que se llamó a la persistencia con la entidad de dominio User
    expect(mockUserQueries.save).toHaveBeenCalled();
  });
});
```

---

## 2. Domain Entity Unit Test (`src/features/users/tests/User.spec.ts`)

Domain entity tests verify business rules and invariant enforcement in complete isolation.

```ts
import { describe, it, expect } from "vitest";
import { User } from "../User.ts";

describe("User Domain Entity", () => {
  it("debe fallar al crear un usuario con un nombre menor a 3 caracteres", () => {
    const result = User.create("Ab", "test@example.com", "uuid-123");

    expect(result.ok).toBe(false);
    if (!result.ok) {
      expect(result.error.statusCode).toBe(400);
      expect(result.error.message).toContain("El nombre debe tener al menos 3 caracteres.");
    }
  });

  it("debe instanciar correctamente un usuario válido con estado activo por defecto", () => {
    const result = User.create("Alice Smith", "alice@example.com", "uuid-123");

    expect(result.ok).toBe(true);
    if (result.ok) {
      const primitives = result.value.toPrimitives();
      expect(primitives.id).toBe("uuid-123");
      expect(primitives.name).toBe("Alice Smith");
      expect(primitives.email).toBe("alice@example.com");
      expect(primitives.isActive).toBe(true);
    }
  });

  it("debe permitir reconstituir un usuario con propiedades existentes sin aplicar validaciones de creación", () => {
    // Reconstitute permite nombres cortos o estados inactivos si ya estaban guardados
    const reconstituted = User.reconstitute({
      id: "uuid-123",
      name: "Ab", // Nombre corto (que fallaría en create)
      email: "test@example.com",
      isActive: false, // Inactivo
    });

    expect(reconstituted).toBeInstanceOf(User);

    const primitives = reconstituted.toPrimitives();
    expect(primitives.id).toBe("uuid-123");
    expect(primitives.name).toBe("Ab");
    expect(primitives.email).toBe("test@example.com");
    expect(primitives.isActive).toBe(false);
  });

  it("debe retornar una copia correcta del estado interno al llamar a toPrimitives", () => {
    const userResult = User.create("John Doe", "john@example.com", "uuid-999");
    expect(userResult.ok).toBe(true);

    if (userResult.ok) {
      const user = userResult.value;
      const primitives1 = user.toPrimitives();
      const primitives2 = user.toPrimitives();

      // Debe ser el mismo contenido pero diferente referencia de objeto
      expect(primitives1).toEqual(primitives2);
      expect(primitives1).not.toBe(primitives2);
    }
  });
});
```

### Key Testing Patterns for Domain Entities

1. **Test Factory Methods**: Verify `create()` enforces invariants and returns `Result`
2. **Test Reconstitution**: Verify `reconstitute()` doesn't apply creation validations
3. **Test Immutability**: Verify `toPrimitives()` returns new object references
4. **Test Business Methods**: If entity has domain methods (like `rename()`), test they enforce rules

---

## 3. Route Integration Test (`src/features/users/tests/users.routes.spec.ts`)

This test maps Hono route mappings, Zod middleware validation (`zValidator`), and standard HTTP responses. We bypass the database entirely by mocking the container's use cases.

```ts
import { describe, it, expect, vi } from "vitest";
import { Hono } from "hono";
import { makeUserRoutes } from "../users.routes.ts";
import { ok, fail } from "@shared/result.ts";
import { AppError } from "@shared/errors/AppError.ts";

describe("Users HTTP Routes", () => {
  it("should return 400 Bad Request if validation fails", async () => {
    // 1. Create a dummy instance of routes with mocked use cases
    const mockCreateUser = {
      execute: vi.fn(async () => fail(AppError.BadRequest("Validation Failed"))),
    } as any;
    
    const mockGetList = { execute: vi.fn(async () => []) } as any;
    const mockGetById = { execute: vi.fn(async () => null) } as any;

    const app = new Hono();
    app.route("/users", makeUserRoutes(mockCreateUser, mockGetList, mockGetById));

    // 2. Dispatch a POST request with invalid JSON parameters
    const response = await app.request("/users", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        name: "J", // Too short (Zod validation will trigger)
        email: "invalid-email",
      }),
    });

    expect(response.status).toBe(400);
    const json = await response.json();
    expect(json.errors).toBeDefined(); // Formatted validation errors object
  });

  it("should return 201 Created on successful registration", async () => {
    const mockCreateUser = {
      execute: vi.fn(async () => ok({
        id: "user-uuid",
        name: "Alice Smith",
        email: "alice@example.com",
        isActive: true,
      })),
    } as any;
    
    const mockGetList = { execute: vi.fn(async () => []) } as any;
    const mockGetById = { execute: vi.fn(async () => null) } as any;

    const app = new Hono();
    app.route("/users", makeUserRoutes(mockCreateUser, mockGetList, mockGetById));

    const response = await app.request("/users", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        name: "Alice Smith",
        email: "alice@example.com",
      }),
    });

    expect(response.status).toBe(201); // 201 Created status code mapping
    const json = await response.json();
    expect(json.name).toBe("Alice Smith");
    expect(json.email).toBe("alice@example.com");
  });
});
```

---

## 4. Database Integration Test (`src/features/users/tests/users.queries.spec.ts`)

Database tests execute actual statements against your PostgreSQL test cluster. To keep integration tests isolated, perform each test within a Drizzle transaction and intentionally roll it back at the end.

```ts
import { describe, it, expect, beforeAll, afterAll } from "vitest";
import { drizzle } from "drizzle-orm/postgres-js";
import postgres from "postgres";
import { envConfig } from "../../env.config.ts";
import { makeUserQueries } from "../users.queries.ts";
import { usersTable } from "@db/schemas/index.ts";
import { User } from "../User.ts";

describe("Users Database Queries", () => {
  let sql: postgres.Sql;
  let db: any;

  beforeAll(() => {
    // Connect to the test PostgreSQL instance
    sql = postgres(envConfig.DATABASE_URL, { max: 1 });
    db = drizzle(sql);
  });

  afterAll(async () => {
    // Close the postgres connection pool
    await sql.end();
  });

  it("should persist and read a user record in a rollback block", async () => {
    // Execute all assertions in a self-contained transaction
    await db.transaction(async (tx: any) => {
      const queries = makeUserQueries(tx);
      const userResult = User.create("Database Tester", "db@test.com", "d88bcf36-58df-4161-9c60-84cf70b135bb");
      
      expect(userResult.ok).toBe(true);
      const user = userResult.value!;

      // 1. Perform save
      await queries.save(user);

      // 2. Fetch record
      const foundUser = await queries.findById(user.toPrimitives().id);

      expect(foundUser).not.toBeNull();
      expect(foundUser!.toPrimitives().name).toBe("Database Tester");
      expect(foundUser!.toPrimitives().email).toBe("db@test.com");

      // 3. Rollback explicitly to leave the database clean
      tx.rollback();
    });
  });
});
```
