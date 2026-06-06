# Response Patterns

This guide covers HTTP response formatting, the `respond()` helper, and status code conventions.

## The respond() Helper

`src/shared/respond.ts`:

```ts
import { Context } from 'hono'
import type { Result } from "./result";
import type { ContentfulStatusCode } from "hono/utils/http-status";
import type { ErrorResponse } from "@shared/errors/ErrorTypes.ts";

export const respond = <T>(
  c: Context,
  result: Result<T, ErrorResponse>,
  statusCode = 200
): Response =>
  result.ok
    ? c.json(result.value, statusCode as ContentfulStatusCode)
    : c.json({ error: result.error.message }, result.error.statusCode as ContentfulStatusCode)
```

## When to Use respond()

Use `respond()` for use cases that return `Result<T, ErrorResponse>`:

```ts
router.post('/', zValidator('json', CreateUserSchema, validationErrorAsJson), async (c) => {
  const body = c.req.valid("json");

  const result = await createUser.execute(body);
  return respond(c, result, 201); // 201 for created resources
});

router.get('/:id', zValidator('param', getByIdSchema, validationErrorAsJson), async (c) => {
  const { id } = c.req.valid('param');

  const result = await getById.execute(id);
  return respond(c, result); // defaults to 200
});
```

## When to Use c.json()

Use direct `c.json()` for use cases that return plain values (not `Result`):

```ts
router.get('/list', async (c) => {
  const result = await getList.execute();
  return c.json(result, 200);
});
```

## Status Code Conventions

### Success Codes

- **200 OK** - Successful GET, PUT, PATCH, or DELETE
- **201 Created** - Successful POST that creates a resource
- **204 No Content** - Successful DELETE with no response body

```ts
// 200 - Get resource
router.get('/:id', async (c) => {
  const result = await getById.execute(id);
  return respond(c, result); // defaults to 200
});

// 201 - Create resource
router.post('/', async (c) => {
  const result = await createUser.execute(body);
  return respond(c, result, 201);
});

// 200 - Update resource
router.patch('/:id', async (c) => {
  const result = await updateUser.execute(id, body);
  return respond(c, result); // 200
});

// 204 - Delete resource (no body)
router.delete('/:id', async (c) => {
  await deleteUser.execute(id);
  return c.body(null, 204);
});
```

### Error Codes

Error codes are set in `AppError` helpers:

- **400 Bad Request** - Invalid input or business rule violation
- **401 Unauthorized** - Missing or invalid authentication
- **403 Forbidden** - Authenticated but lacks permission
- **404 Not Found** - Resource doesn't exist
- **409 Conflict** - Duplicate resource or state conflict
- **500 Internal Server Error** - Unexpected errors

## Success Response Patterns

### Single Resource

```ts
// GET /users/123
router.get('/:id', async (c) => {
  const result = await getUserById.execute(id);
  return respond(c, result);
});

// Response body:
{
  "id": "123",
  "name": "John Doe",
  "email": "john@example.com"
}
```

### Created Resource

```ts
// POST /users
router.post('/', async (c) => {
  const result = await createUser.execute(body);
  return respond(c, result, 201);
});

// Response body:
{
  "id": "new-uuid",
  "name": "Jane Doe",
  "email": "jane@example.com",
  "isActive": true
}
```

### List of Resources

```ts
// GET /users/list
router.get('/list', async (c) => {
  const result = await getList.execute();
  return c.json(result, 200);
});

// Response body:
[
  { "id": "1", "name": "John", "email": "john@example.com" },
  { "id": "2", "name": "Jane", "email": "jane@example.com" }
]
```

### Operation Result

```ts
// POST /auth/login
router.post('/login', async (c) => {
  const result = await login.execute(email, password);
  return respond(c, result);
});

// Response body:
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

## Error Response Patterns

### Validation Error (400)

Returned by `validationErrorAsJson`:

```json
{
  "errors": {
    "name": "String must contain at least 3 character(s)",
    "email": "Invalid email"
  }
}
```

### Business Logic Error (400)

Returned by `respond()` when use case returns `fail(AppError.BadRequest(...))`:

```json
{
  "error": "El nombre debe tener al menos 3 caracteres."
}
```

### Not Found (404)

Returned by `respond()` when use case returns `fail(AppError.NotFound(...))`:

```json
{
  "error": "User Not Found"
}
```

### Conflict (409)

Returned by `respond()` when use case returns `fail(AppError.Conflict(...))`:

```json
{
  "error": "No se puede registar un email duplicado."
}
```

### Unauthorized (401)

Returned by auth middleware when JWT is invalid:

```json
{
  "error": "Unauthorized"
}
```

### Internal Server Error (500)

Returned by global error handler for unexpected errors:

```json
{
  "error": "Internal server error"
}
```

## Pagination Pattern

For paginated lists:

```ts
export interface PaginatedResponse<T> {
  data: T[];
  pagination: {
    page: number;
    limit: number;
    total: number;
    totalPages: number;
  };
}

router.get('/products', zValidator('query', PaginationQuerySchema), async (c) => {
  const query = c.req.valid('query');

  const result = await listProducts.execute(query);
  return c.json(result, 200);
});

// Response:
{
  "data": [
    { "id": "1", "name": "Product 1" },
    { "id": "2", "name": "Product 2" }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 100,
    "totalPages": 5
  }
}
```

## Custom Headers

Add custom headers when needed:

```ts
router.get('/download/:id', async (c) => {
  const file = await getFile.execute(id);

  c.header('Content-Type', 'application/pdf');
  c.header('Content-Disposition', `attachment; filename="${file.name}"`);

  return c.body(file.data);
});
```

## Response Transformations

Keep transformations in use cases or queries, not routes:

```ts
// ❌ Bad - transformation in route
router.get('/users/:id', async (c) => {
  const user = await getUserById.execute(id);

  // Don't transform here
  const transformed = {
    id: user.id,
    fullName: `${user.firstName} ${user.lastName}`,
  };

  return c.json(transformed);
});

// ✅ Good - transformation in use case
export class GetUserById {
  async execute(id: string): Promise<Result<UserResponse>> {
    const user = await this.userQueries.findById(id);

    if (!user) {
      return userNotFoundError();
    }

    // Transform in use case
    const { id, firstName, lastName } = user.toPrimitives();
    return ok({
      id,
      fullName: `${firstName} ${lastName}`,
    });
  }
}
```

## Best Practices

1. **Use `respond()` for Result values** - handles success/failure automatically
2. **Use `c.json()` for plain values** - when use case doesn't return Result
3. **Set correct status codes** - 201 for creates, 200 for other success
4. **Keep routes thin** - no business logic or transformations
5. **Use typed responses** - define response interfaces in use cases
6. **Don't expose internal fields** - use case responses should hide sensitive data
7. **Consistent error format** - let `respond()` and error handlers format errors
8. **Use pagination for lists** - when lists can grow large
9. **Document response shapes** - in use case or schema files
10. **Test response contracts** - integration tests should verify response structure
