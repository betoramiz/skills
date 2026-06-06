---
name: ts-routing
description: Hono routing, middleware, validation, and HTTP layer patterns for the web-api TypeScript backend. Use when creating routes, applying middleware (auth, logging), implementing validation with Zod, or wiring HTTP endpoints to use cases.
---

# TS Routing & Middleware

## Overview

Use this skill when working with Hono routes, middleware (authentication, logging), request validation, and HTTP response formatting in the `web-api` project.

## Workflow

1. Routes are factory functions that receive use case instances as parameters
2. Apply middleware using `router.use()` before defining endpoints
3. Use `zValidator` for request validation (body, params, query)
4. Use `respond()` helper for use cases that return `Result<T, ErrorResponse>`
5. Use direct `c.json()` for use cases that return plain values
6. Keep HTTP concerns (status codes, headers) in routes, not use cases
7. Wire new routes in the feature container and mount in `src/index.ts`

## Common Patterns

### Route Factory with Middleware

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

  // Apply middleware to all routes
  router.use('*', authMiddleware);
  router.use(pinoLogger({ pino: appLogger }));

  // Define endpoints
  router.post('/', zValidator('json', CreateUserSchema, validationErrorAsJson), async (c) => {
    const body = c.req.valid("json");
    const result = await createUser.execute(body);
    return respond(c, result, 201);
  });

  router.get('/list', async (c) => {
    const result = await getList.execute();
    return c.json(result, 200);
  });

  router.get('/:id', zValidator('param', getByIdSchema, validationErrorAsJson), async (c) => {
    const { id } = c.req.valid('param');
    const result = await getById.execute(id);
    return respond(c, result);
  });

  return router;
};
```

### Public Routes (No Auth)

For public endpoints like authentication:

```ts
export const makeAuthRoutes = (register: RegisterUseCase, login: LoginUseCase) => {
  const router = new Hono();

  // Only apply logging, no auth middleware
  router.use(pinoLogger({ pino: appLogger }));

  router.post('/register', zValidator('json', registerSchema, validationErrorAsJson), async c => {
    const body = c.req.valid('json');
    const result = await register.execute(body);
    return respond(c, result);
  });

  router.post('/login', zValidator('json', loginSchema), async c => {
    const { email, password } = c.req.valid('json');
    const result = await login.execute(email, password);
    return respond(c, result);
  });

  return router;
};
```

### Validation Patterns

Use `zValidator` with three parameters:
1. Location: `'json'`, `'param'`, `'query'`, `'header'`
2. Schema: Zod schema
3. Error handler: `validationErrorAsJson`

```ts
// Body validation
router.post('/',
  zValidator('json', CreateSchema, validationErrorAsJson),
  async (c) => {
    const body = c.req.valid("json");
    // ...
  }
);

// Param validation
router.get('/:id',
  zValidator('param', getByIdSchema, validationErrorAsJson),
  async (c) => {
    const { id } = c.req.valid('param');
    // ...
  }
);

// Multiple validators
router.patch('/:id',
  zValidator('param', getByIdSchema, validationErrorAsJson),
  zValidator('json', UpdateSchema, validationErrorAsJson),
  async (c) => {
    const { id } = c.req.valid('param');
    const body = c.req.valid('json');
    // ...
  }
);
```

## References

- Read `references/middleware-guide.md` for authentication, logging, and custom middleware patterns
- Read `references/validation-guide.md` for Zod schema patterns and validation error handling
- Read `references/response-patterns.md` for using the `respond()` helper and formatting responses

## Checks

Before finishing a routing change:

- Confirm middleware is applied in correct order (auth before routes that need it)
- Confirm `zValidator` uses `validationErrorAsJson` for consistent error formatting
- Confirm routes use `respond()` for `Result` values and `c.json()` for plain values
- Confirm route handlers are thin and delegate to use cases
- Confirm new routes are mounted in `src/index.ts` via the feature container
- Confirm protected routes have `authMiddleware` applied
- Run `bun run dev` to verify routes work correctly
