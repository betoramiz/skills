# Validation Guide

This guide covers Zod schema patterns, request validation, and error handling for the Hono API.

## Zod Schema Patterns

### Basic Schema

Define schemas in `<feature>.schemas.ts`:

```ts
import { z } from "zod";

export const CreateUserSchema = z.object({
  name: z.string().min(3),
  email: z.string().email(),
});

export type CreateUserCommand = z.infer<typeof CreateUserSchema>;
```

### Complex Validations

```ts
export const CreateProductSchema = z.object({
  name: z.string().min(3).max(100),
  sku: z.string().regex(/^[A-Z0-9-]+$/),
  priceInCents: z.number().int().min(0).max(1000000),
  categoryId: z.uuid(),
  tags: z.array(z.string()).optional(),
  metadata: z.record(z.string()).optional(),
});

export type CreateProductCommand = z.infer<typeof CreateProductSchema>;
```

### Param Validation

```ts
export const getByIdSchema = z.object({
  id: z.uuid(),
});

export type GetByIdParams = z.infer<typeof getByIdSchema>;
```

### Query Param Validation

```ts
export const ListProductsQuerySchema = z.object({
  page: z.coerce.number().int().min(1).default(1),
  limit: z.coerce.number().int().min(1).max(100).default(20),
  category: z.string().optional(),
  sortBy: z.enum(['name', 'price', 'createdAt']).default('createdAt'),
  order: z.enum(['asc', 'desc']).default('desc'),
});

export type ListProductsQuery = z.infer<typeof ListProductsQuerySchema>;
```

### Nested Objects

```ts
export const AddressSchema = z.object({
  street: z.string().min(1),
  city: z.string().min(1),
  state: z.string().length(2),
  zipCode: z.string().regex(/^\d{5}(-\d{4})?$/),
});

export const CreateOrderSchema = z.object({
  items: z.array(z.object({
    productId: z.uuid(),
    quantity: z.number().int().min(1),
  })).min(1),
  shippingAddress: AddressSchema,
  billingAddress: AddressSchema.optional(),
  notes: z.string().max(500).optional(),
});

export type CreateOrderCommand = z.infer<typeof CreateOrderSchema>;
```

### Conditional Validation

```ts
export const UpdateUserSchema = z.object({
  name: z.string().min(3).optional(),
  email: z.string().email().optional(),
  password: z.string().min(8).optional(),
  confirmPassword: z.string().optional(),
}).refine(
  (data) => {
    if (data.password && !data.confirmPassword) return false;
    if (data.password !== data.confirmPassword) return false;
    return true;
  },
  {
    message: "Passwords must match",
    path: ["confirmPassword"],
  }
);
```

## Using Validation in Routes

### Body Validation

```ts
import { zValidator } from "@hono/zod-validator";
import { validationErrorAsJson } from "@shared/validation-errors.ts";

router.post('/',
  zValidator('json', CreateUserSchema, validationErrorAsJson),
  async (c) => {
    const body = c.req.valid("json");
    // body is fully typed as CreateUserCommand
    const result = await createUser.execute(body);
    return respond(c, result, 201);
  }
);
```

### Param Validation

```ts
router.get('/:id',
  zValidator('param', getByIdSchema, validationErrorAsJson),
  async (c) => {
    const { id } = c.req.valid('param');
    // id is typed as string (validated UUID)
    const result = await getById.execute(id);
    return respond(c, result);
  }
);
```

### Query Param Validation

```ts
router.get('/search',
  zValidator('query', SearchQuerySchema, validationErrorAsJson),
  async (c) => {
    const query = c.req.valid('query');
    // query is fully typed with defaults applied
    const result = await searchProducts.execute(query);
    return c.json(result);
  }
);
```

### Multiple Validators

```ts
router.patch('/:id',
  zValidator('param', getByIdSchema, validationErrorAsJson),
  zValidator('json', UpdateUserSchema, validationErrorAsJson),
  async (c) => {
    const { id } = c.req.valid('param');
    const body = c.req.valid('json');

    const result = await updateUser.execute(id, body);
    return respond(c, result);
  }
);
```

## Validation Error Handling

### The validationErrorAsJson Helper

`src/shared/validation-errors.ts`:

```ts
import type { Context } from "hono";
import type { ZodError } from "zod";

export const validationErrorAsJson = (result: any, c: Context) => {
  if (!result.success) {
    const error = result.error as ZodError;
    const errors: Record<string, string> = {};

    error.errors.forEach((err) => {
      const path = err.path.join(".");
      errors[path] = err.message;
    });

    return c.json({
      errors,
    }, 400);
  }
};
```

### Validation Error Response Format

When validation fails, the response looks like:

```json
{
  "errors": {
    "name": "String must contain at least 3 character(s)",
    "email": "Invalid email",
    "priceInCents": "Number must be greater than or equal to 0"
  }
}
```

### Custom Error Messages

```ts
export const CreateUserSchema = z.object({
  name: z.string()
    .min(3, "Name must be at least 3 characters")
    .max(100, "Name cannot exceed 100 characters"),
  email: z.string()
    .email("Please provide a valid email address"),
  age: z.number()
    .int("Age must be a whole number")
    .min(18, "You must be at least 18 years old")
    .max(120, "Age must be realistic"),
});
```

## Authentication Schema Patterns

### Register Schema

```ts
export const registerSchema = z.object({
  email: z.string().email("Invalid email format"),
  password: z.string()
    .min(6, "Password must be at least 6 characters")
    .max(100, "Password is too long"),
  confirmPassword: z.string(),
}).refine(
  (data) => data.password === data.confirmPassword,
  {
    message: "Passwords do not match",
    path: ["confirmPassword"],
  }
);

export type RegisterCommand = z.infer<typeof registerSchema>;
```

### Login Schema

```ts
export const loginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(1, "Password is required"),
});

export type LoginCommand = z.infer<typeof loginSchema>;
```

## Advanced Patterns

### Transform and Parse

```ts
export const CreateProductSchema = z.object({
  name: z.string().transform(val => val.trim()),
  sku: z.string().transform(val => val.toUpperCase()),
  priceInCents: z.number().int().min(0),
  tags: z.string()
    .transform(val => val.split(',').map(t => t.trim()))
    .optional(),
});
```

### Discriminated Unions

```ts
export const PaymentMethodSchema = z.discriminatedUnion('type', [
  z.object({
    type: z.literal('credit_card'),
    cardNumber: z.string().regex(/^\d{16}$/),
    cvv: z.string().regex(/^\d{3,4}$/),
    expiryDate: z.string().regex(/^\d{2}\/\d{2}$/),
  }),
  z.object({
    type: z.literal('paypal'),
    email: z.string().email(),
  }),
  z.object({
    type: z.literal('bank_transfer'),
    accountNumber: z.string(),
    routingNumber: z.string(),
  }),
]);

export type PaymentMethod = z.infer<typeof PaymentMethodSchema>;
```

### Partial Schemas for Updates

```ts
export const CreateUserSchema = z.object({
  name: z.string().min(3),
  email: z.string().email(),
  role: z.enum(['user', 'admin']),
});

// All fields optional for partial updates
export const UpdateUserSchema = CreateUserSchema.partial();

export type UpdateUserCommand = z.infer<typeof UpdateUserSchema>;
```

## Best Practices

1. **Define schemas in `<feature>.schemas.ts`** - keep validation separate from routes
2. **Export inferred types** - use `z.infer<>` for type-safe commands
3. **Use custom error messages** - make validation failures user-friendly
4. **Always use `validationErrorAsJson`** - ensures consistent error format
5. **Validate early** - use zValidator before route logic
6. **Don't duplicate validation** - domain entities validate business rules, Zod validates HTTP input
7. **Use `.transform()`** - for normalizing input (trim, uppercase, etc.)
8. **Use `.default()`** - for optional fields with defaults
9. **Use `.refine()`** - for cross-field validation
10. **Keep schemas simple** - complex business logic belongs in domain entities
