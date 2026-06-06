# Middleware Guide

This guide covers authentication, logging, and custom middleware patterns in the Hono-based web API.

## Authentication Middleware

### JWT Authentication

The project uses Hono's built-in JWT middleware for authentication.

`src/shared/middleware/auth.middleware.ts`:

```ts
import { jwt } from 'hono/jwt';
import { envConfig } from "../../env.config.ts";

export const authMiddleware = jwt({
  secret: envConfig.SECRET,
  alg: "HS256"
});
```

### Applying Auth Middleware

Apply to all routes in a feature:

```ts
export const makeUserRoutes = (...useCases) => {
  const router = new Hono();

  // Protect all user routes
  router.use('*', authMiddleware);

  // All routes below require valid JWT
  router.post('/', ...);
  router.get('/:id', ...);

  return router;
};
```

Apply to specific routes only:

```ts
export const makeProductRoutes = (...useCases) => {
  const router = new Hono();

  // Public route - no auth needed
  router.get('/list', async (c) => {
    // ...
  });

  // Protected routes - auth required
  router.use('/:id/*', authMiddleware);
  router.post('/', authMiddleware, async (c) => {
    // ...
  });

  return router;
};
```

### Accessing JWT Payload

When authMiddleware is applied, access the JWT payload via context:

```ts
router.get('/me', authMiddleware, async (c) => {
  const payload = c.get('jwtPayload');
  const userId = payload.sub; // User ID from token

  const result = await getUserById.execute(userId);
  return respond(c, result);
});
```

## Logging Middleware

### Pino Logger

The project uses Pino for structured logging via `hono-pino`.

`src/shared/middleware/logger.middleware.ts`:

```ts
import pino from "pino";

export const appLogger = pino({
  level: "info",
  transport: {
    target: "pino-pretty",
    options: {
      colorize: true,
    },
  },
});
```

### Applying Logger

Apply logger to feature routes:

```ts
import { pinoLogger } from "hono-pino";
import { appLogger } from "@shared/middleware/logger.middleware.ts";

export const makeUserRoutes = (...) => {
  const router = new Hono();

  router.use(pinoLogger({ pino: appLogger }));

  // Routes will automatically log requests
  router.post('/', ...);

  return router;
};
```

### Custom Logging in Routes

Access the logger in route handlers:

```ts
router.post('/complex-operation', async (c) => {
  const logger = c.get('logger');

  logger.info({ userId: 'abc' }, 'Starting complex operation');

  try {
    const result = await complexUseCase.execute();
    logger.info('Complex operation succeeded');
    return respond(c, result);
  } catch (error) {
    logger.error({ error }, 'Complex operation failed');
    throw error;
  }
});
```

## Middleware Order

Apply middleware in this order:

1. **Logger first** - to log all requests including auth failures
2. **Auth middleware** - to authenticate requests
3. **Route-specific middleware** - any custom middleware
4. **Route handlers** - the actual endpoints

```ts
export const makeRoutes = (...) => {
  const router = new Hono();

  // 1. Logging
  router.use(pinoLogger({ pino: appLogger }));

  // 2. Authentication (if needed)
  router.use('*', authMiddleware);

  // 3. Custom middleware (if any)
  router.use(customMiddleware);

  // 4. Routes
  router.post('/', ...);

  return router;
};
```

## Custom Middleware

### Creating Custom Middleware

```ts
// src/shared/middleware/rate-limit.middleware.ts
import { createMiddleware } from 'hono/factory';

const requestCounts = new Map<string, number>();

export const rateLimitMiddleware = createMiddleware(async (c, next) => {
  const ip = c.req.header('x-forwarded-for') || 'unknown';

  const count = requestCounts.get(ip) || 0;
  if (count > 100) {
    return c.json({ error: 'Rate limit exceeded' }, 429);
  }

  requestCounts.set(ip, count + 1);

  await next();
});
```

### Applying Custom Middleware

```ts
import { rateLimitMiddleware } from "@shared/middleware/rate-limit.middleware.ts";

export const makePublicRoutes = (...) => {
  const router = new Hono();

  router.use(pinoLogger({ pino: appLogger }));
  router.use(rateLimitMiddleware);

  router.post('/signup', ...);

  return router;
};
```

## CORS Middleware

If you need CORS support:

```ts
import { cors } from 'hono/cors';

const app = new Hono();

app.use('*', cors({
  origin: ['http://localhost:3000', 'https://yourapp.com'],
  allowMethods: ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS'],
  allowHeaders: ['Content-Type', 'Authorization'],
  credentials: true,
}));
```

## Error Handling in Middleware

Middleware errors are caught by the global error handler in `src/app.ts`:

```ts
app.onError((error, c) => {
  const dbError = parseDbError(error)
  if (dbError) {
    return c.json({
      error: dbError.message,
      code: dbError.statusCode,
    }, dbError.statusCode as ContentfulStatusCode)
  } else if (error instanceof HTTPException){
    return c.json({ error: error.message }, error.status);
  }
  console.error('Unexpected error:', error)
  return c.json({ error: 'Internal server error' }, 500)
})
```

Middleware can throw `HTTPException` for expected errors:

```ts
import { HTTPException } from "hono/http-exception";

export const customMiddleware = createMiddleware(async (c, next) => {
  const header = c.req.header('x-custom-header');

  if (!header) {
    throw new HTTPException(400, { message: 'Missing required header' });
  }

  await next();
});
```

## Best Practices

1. **Apply logger first** - ensures all requests are logged
2. **Auth after logger** - logs failed auth attempts
3. **Keep middleware thin** - complex logic belongs in use cases
4. **Use HTTPException** - for expected middleware failures
5. **Let unexpected errors bubble** - caught by global handler
6. **Don't duplicate validation** - use `zValidator` in routes instead
7. **Avoid side effects** - middleware should enhance context, not mutate data
