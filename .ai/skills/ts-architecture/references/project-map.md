# Project Map

## Runtime and Frameworks

The `web-api` project is a Bun TypeScript backend.

- Runtime: Bun.
- HTTP framework: Hono.
- Request validation: Zod with `@hono/zod-validator`.
- Database access: Drizzle ORM over `postgres`.
- Logging: Pino via `hono-pino` middleware.
- Authentication: JWT via `hono/jwt`.
- Password hashing: bcrypt.
- Error shape: `ErrorResponse` from `src/shared/errors/ErrorTypes.ts`.
- Application result shape: `Result<T, E>` from `src/shared/result.ts`.
- Path aliases: `@db/*` -> `src/db/*`, `@shared/*` -> `src/shared/*`.
- Import style: explicit `.ts` extensions for local TypeScript imports.

## Entry Points

`src/index.ts` imports the shared Hono app, mounts feature routes, and exports the Bun server object.

```ts
import app from './app';
import { envConfig } from "./env.config.ts";
import userRoutes from "./features/users/users.container.ts";
import authRoutes from "./features/auth/auth.container.ts";

app.route('/users', userRoutes);
app.route('/auth', authRoutes);

export default {
  port: envConfig.PORT,
  fetch: app.fetch,
};
```

Use this file to mount new route groups:

```ts
import productRoutes from "./features/products/products.container.ts";

app.route('/products', productRoutes);
```

`src/app.ts` owns the Hono instance and global error handling. It maps known Postgres errors with `parseDbError`, handles HTTPException, and returns a generic `500` for unexpected errors.

## Environment Configuration

`src/env.config.ts` validates environment variables with Zod:

```ts
const envSchema = z.object({
  DATABASE_URL: z.string().url(),
  PORT: z.coerce.number().default(3000),
  NODE_ENV: z.enum(['development', 'production', 'test']).default('development'),
  SECRET: z.string().nonempty().min(28),
});
```

When adding environment variables, add them here and consume the parsed `envConfig` object. Avoid reading `process.env` directly elsewhere.

## Database Layer

The database layer lives under `src/db/`.

- `connection.ts` creates the `postgres` client and Drizzle database instance.
- `index.ts` re-exports the database and schemas.
- `drizzle.config.ts` configures Drizzle Kit for migrations.
- `schemas/*.schema.ts` defines Drizzle table schemas.
- `schemas/index.ts` re-exports all table schemas.
- `schemas/relations.ts` defines Drizzle relational mappings between tables.
- `drizzle/` directory contains generated migration SQL files.

Current table pattern:

```ts
import { pgTable, text, boolean, uuid } from 'drizzle-orm/pg-core';
import { authTable } from "@db/schemas/auth.schema.ts";

export const usersTable = pgTable('users', {
  id: uuid('id').primaryKey(),
  name: text('name').notNull(),
  isActive: boolean('is_active').notNull().default(true),
  authId: uuid('auth_id').references(() => authTable.id).unique(),
});
```

When adding a table:

1. Create `src/db/schemas/<resource>.schema.ts`.
2. Define relationships in `src/db/schemas/relations.ts` if needed.
3. Export the table from `src/db/schemas/index.ts`.
4. Run `bun run db:generate:migration` to generate SQL migration files.
5. Run `bun run db:migrate` to apply migrations.
6. Use the exported table from feature query factories through `@db/schemas`.

## Feature Module Shape

Feature modules live under `src/features/<feature>/`.

Current users module:

```text
src/features/users/
  User.ts
  users.container.ts
  users.queries.ts
  users.routes.ts
  users.schemas.ts
  usecases/
    create.usecase.ts
    errors.ts
    getById.usecase.ts
    list.usecase.ts
  tests/
    User.spec.ts
    usecases/
      create.usecase.spec.ts
      getById.usecase.spec.ts
      list.usecase.spec.ts
```

Current auth module:

```text
src/features/auth/
  Auth.ts
  auth.container.ts
  auth.queries.ts
  auth.routes.ts
  auth.schemas.ts
  auth.dtos.ts
  usecases/
    register.usecase.ts
    login.usecase.ts
    errors.ts
```

Use the same shape for new modules.

## HTTP Route Layer

Routes are factory functions that receive use case instances. They should not create database connections or use cases directly.

Current pattern with middleware:

```ts
import { Hono } from "hono";
import { zValidator } from "@hono/zod-validator";
import { authMiddleware } from "@shared/middleware/auth.middleware.ts";
import { pinoLogger } from "hono-pino";
import { appLogger } from "@shared/middleware/logger.middleware.ts";
import { respond } from "@shared/respond.ts";
import { validationErrorAsJson } from "@shared/validation-errors.ts";

export const makeUserRoutes = (
  createUser: CreateUser,
  getList: GetList,
  getById: GetUserById
) => {
  const router = new Hono();

  router.use('*', authMiddleware);
  router.use(pinoLogger({ pino: appLogger }));

  router.post('/', zValidator('json', CreateUserSchema, validationErrorAsJson), async (c) => {
    const body = c.req.valid("json");
    const result = await createUser.execute(body);
    return respond(c, result);
  });

  router.get('/:id', zValidator('param', getByIdSchema, validationErrorAsJson), async (c) => {
    const { id } = c.req.valid('param');
    const result = await getById.execute(id);
    return respond(c, result);
  });

  return router;
};
```

Use `respond(c, result)` for `Result`-returning use cases. Use direct `c.json(...)` only when the use case intentionally returns a plain value.

## Schemas and Commands

External request shape belongs in `<feature>.schemas.ts`.

```ts
export const CreateUserSchema = z.object({
  name: z.string().min(3),
  email: z.string().email(),
});

export type CreateUserCommand = z.infer<typeof CreateUserSchema>;
```

Use inferred command types in use cases.

## Domain Entity Layer

Entities own invariants and conversion to/from persistence primitives.

```ts
export class User {
  private constructor(private props: UserProps) {}

  public static create(name: string, email: string, id: string): Result<User, ErrorResponse> {
    if (name.length < 3) {
      return fail(AppError.BadRequest("The name must have at least 3 characters."));
    }
    return ok(new User({ id, name, email, isActive: true }));
  }

  public static reconstitute(props: UserProps): User {
    return new User(props);
  }

  public toPrimitives(): UserProps {
    return { ...this.props };
  }
}
```

Use `create` for new instances because it validates invariants. Use `reconstitute` for rows loaded from persistence. Use `toPrimitives` when saving or responding.

## Query Factories

Queries are factories that receive a `Db` instance and return an object of persistence operations.

```ts
import { eq } from "drizzle-orm";
import { type Db } from "@db/connection.ts";
import { usersTable } from "@db/schemas";
import { User } from "./User.ts";

export const makeUserQueries = (database: Db) => ({
  findById: async (id: string): Promise<User | null> => {
    const [row] = await database.select().from(usersTable).where(eq(usersTable.id, id));
    if (!row) return null;

    return User.reconstitute(row);
  },

  save: async (user: User): Promise<void> => {
    await database.insert(usersTable).values(user.toPrimitives());
  },

  getAll: async () => {
    return database.select({
      id: usersTable.id,
      name: usersTable.name,
      email: usersTable.email,
    }).from(usersTable);
  }
});

export type UserQueries = ReturnType<typeof makeUserQueries>;
```

Keep Drizzle details here. Return domain entities for operations that need behavior, and return projections for list/read operations that do not need domain logic.

## Containers

Containers wire dependencies once:

```ts
const userQueries = makeUserQueries(db);
const createUser = new CreateUser(userQueries);
const getUserById = new GetUserById(userQueries);
const getList = new GetList(userQueries);

const routes = makeUserRoutes(createUser, getList, getUserById);

export default routes;
```

When adding a new use case, instantiate it here and pass it to the route factory.

## Shared Primitives

`src/shared/result.ts`:

```ts
export type Result<T, E = ErrorResponse> = Success<T> | Failure<E>
export const ok = <T>(value: T): Success<T> => ({ ok: true, value })
export const fail = <E>(error: E): Failure<E> => ({ ok: false, error })
```

`src/shared/respond.ts` maps a `Result` into a Hono JSON response.

`src/shared/validation-errors.ts` converts Zod validation issues into `{ errors: Record<string, string> }`.

`src/shared/errors/AppError.ts` centralizes expected error creation:

- `BadRequest`
- `NotFound`
- `DatabaseProblem`
- `Conflict`
- `ServiceUnavailable`

`src/shared/db-errors.ts` maps known Postgres errors:

- Unique violation -> conflict.
- Foreign key violation -> bad request.
- Not null violation -> bad request.

## Middleware

`src/shared/middleware/auth.middleware.ts` provides JWT authentication:

```ts
import { jwt } from 'hono/jwt';
import { envConfig } from "../../env.config.ts";

export const authMiddleware = jwt({
  secret: envConfig.SECRET,
  alg: "HS256"
});
```

Apply this middleware to protected routes using `router.use('*', authMiddleware)`.

`src/shared/middleware/logger.middleware.ts` creates a Pino logger instance that can be used with `hono-pino` middleware.
