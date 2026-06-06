---
name: quick-api
description: Quick reference for rapidly building complete API features in this Bun + Hono + Drizzle architecture. Use this as a checklist when creating new features from scratch.
---

# Quick API Development

## Overview

This skill provides a step-by-step checklist for rapidly creating complete, production-ready API features following the project's Clean Architecture patterns.

## Complete Feature Checklist

When adding a new feature (e.g., "products"), follow these steps in order:

### 1. Database Schema

**File**: `src/db/schemas/products.schema.ts`

```ts
import { pgTable, text, boolean, uuid, integer } from 'drizzle-orm/pg-core';

export const productsTable = pgTable('products', {
  id: uuid('id').primaryKey(),
  name: text('name').notNull(),
  sku: text('sku').notNull().unique(),
  priceInCents: integer('price_in_cents').notNull(),
  isActive: boolean('is_active').notNull().default(true),
});
```

**Export in** `src/db/schemas/index.ts`:

```ts
export * from './products.schema.ts';
```

**Generate and run migration**:

```bash
bun run db:generate:migration
bun run db:migrate
```

### 2. Domain Entity

**File**: `src/features/products/Product.ts`

```ts
import type { ErrorResponse } from "@shared/errors/ErrorTypes.ts";
import { AppError } from "@shared/errors/AppError.ts";
import { fail, ok, type Result } from "@shared/result.ts";

export interface ProductProps {
  id: string;
  name: string;
  sku: string;
  priceInCents: number;
  isActive: boolean;
}

export class Product {
  private constructor(private props: ProductProps) {}

  public static create(
    name: string,
    sku: string,
    priceInCents: number,
    id: string
  ): Result<Product, ErrorResponse> {
    if (name.trim().length < 3) {
      return fail(AppError.BadRequest("Product name must have at least 3 characters."));
    }

    if (priceInCents < 0) {
      return fail(AppError.BadRequest("Product price cannot be negative."));
    }

    return ok(new Product({
      id,
      name: name.trim(),
      sku: sku.trim(),
      priceInCents,
      isActive: true,
    }));
  }

  public static reconstitute(props: ProductProps): Product {
    return new Product(props);
  }

  public toPrimitives(): ProductProps {
    return { ...this.props };
  }
}
```

### 3. Request Schemas

**File**: `src/features/products/products.schemas.ts`

```ts
import { z } from "zod";

export const CreateProductSchema = z.object({
  name: z.string().min(3),
  sku: z.string().min(3),
  priceInCents: z.number().int().min(0),
});

export type CreateProductCommand = z.infer<typeof CreateProductSchema>;

export const getProductByIdSchema = z.object({
  id: z.uuid(),
});
```

### 4. Query Factory

**File**: `src/features/products/products.queries.ts`

```ts
import { eq } from "drizzle-orm";
import { type Db } from "@db/connection.ts";
import { productsTable } from "@db/schemas";
import { Product } from "./Product.ts";

export const makeProductQueries = (database: Db) => ({
  save: async (product: Product): Promise<void> => {
    await database.insert(productsTable).values(product.toPrimitives());
  },

  findById: async (id: string): Promise<Product | null> => {
    const [row] = await database
      .select()
      .from(productsTable)
      .where(eq(productsTable.id, id));

    if (!row) return null;
    return Product.reconstitute(row);
  },

  getAll: async () => {
    return database.select({
      id: productsTable.id,
      name: productsTable.name,
      sku: productsTable.sku,
      priceInCents: productsTable.priceInCents,
    }).from(productsTable);
  },
});

export type ProductQueries = ReturnType<typeof makeProductQueries>;
```

### 5. Error Helpers

**File**: `src/features/products/usecases/errors.ts`

```ts
import { AppError } from "@shared/errors/AppError.ts";
import { fail } from "@shared/result.ts";

export const productNotFoundError = (message?: string) =>
  fail(AppError.NotFound(message ?? 'Product Not Found'));
```

### 6. Use Cases

**File**: `src/features/products/usecases/create.usecase.ts`

```ts
import { v4 as uuidv4 } from "uuid";
import { Product, type ProductProps } from "../Product.ts";
import type { ProductQueries } from "../products.queries.ts";
import type { CreateProductCommand } from "../products.schemas.ts";
import type { ErrorResponse } from "@shared/errors/ErrorTypes.ts";
import { fail, ok, type Result } from "@shared/result.ts";

export class CreateProduct {
  constructor(private readonly productQueries: ProductQueries) {}

  async execute(command: CreateProductCommand): Promise<Result<ProductProps, ErrorResponse>> {
    const productResult = Product.create(
      command.name,
      command.sku,
      command.priceInCents,
      uuidv4()
    );

    if (!productResult.ok) {
      return fail(productResult.error);
    }

    const product = productResult.value;
    await this.productQueries.save(product);

    return ok(product.toPrimitives());
  }
}
```

**File**: `src/features/products/usecases/getById.usecase.ts`

```ts
import { ok, type Result } from "@shared/result.ts";
import type { ErrorResponse } from "@shared/errors/ErrorTypes.ts";
import type { ProductQueries } from "../products.queries.ts";
import { productNotFoundError } from "./errors.ts";

export interface GetProductByIdResponse {
  id: string;
  name: string;
  sku: string;
  priceInCents: number;
}

export class GetProductById {
  constructor(private readonly productQueries: ProductQueries) {}

  async execute(productId: string): Promise<Result<GetProductByIdResponse, ErrorResponse>> {
    const product = await this.productQueries.findById(productId);

    if (product === null) {
      return productNotFoundError();
    }

    const { id, name, sku, priceInCents } = product.toPrimitives();
    return ok({ id, name, sku, priceInCents });
  }
}
```

**File**: `src/features/products/usecases/list.usecase.ts`

```ts
import type { ProductQueries } from "../products.queries.ts";

export interface ListProductsResponse {
  id: string;
  name: string;
  sku: string;
  priceInCents: number;
}

export class ListProducts {
  constructor(private readonly productQueries: ProductQueries) {}

  async execute(): Promise<ListProductsResponse[]> {
    return await this.productQueries.getAll();
  }
}
```

### 7. Routes

**File**: `src/features/products/products.routes.ts`

```ts
import { Hono } from "hono";
import { zValidator } from "@hono/zod-validator";
import { authMiddleware } from "@shared/middleware/auth.middleware.ts";
import { pinoLogger } from "hono-pino";
import { appLogger } from "@shared/middleware/logger.middleware.ts";
import { respond } from "@shared/respond.ts";
import { validationErrorAsJson } from "@shared/validation-errors.ts";
import { CreateProductSchema, getProductByIdSchema } from "./products.schemas.ts";
import { CreateProduct } from "./usecases/create.usecase.ts";
import { GetProductById } from "./usecases/getById.usecase.ts";
import { ListProducts } from "./usecases/list.usecase.ts";

export const makeProductRoutes = (
  createProduct: CreateProduct,
  listProducts: ListProducts,
  getProductById: GetProductById
) => {
  const router = new Hono();

  router.use('*', authMiddleware);
  router.use(pinoLogger({ pino: appLogger }));

  router.post('/', zValidator('json', CreateProductSchema, validationErrorAsJson), async (c) => {
    const body = c.req.valid("json");
    const result = await createProduct.execute(body);
    return respond(c, result, 201);
  });

  router.get('/list', async (c) => {
    const result = await listProducts.execute();
    return c.json(result, 200);
  });

  router.get('/:id', zValidator('param', getProductByIdSchema, validationErrorAsJson), async (c) => {
    const { id } = c.req.valid('param');
    const result = await getProductById.execute(id);
    return respond(c, result);
  });

  return router;
};
```

### 8. Container

**File**: `src/features/products/products.container.ts`

```ts
import { db } from "@db/connection.ts";
import { makeProductQueries } from "./products.queries.ts";
import { CreateProduct } from "./usecases/create.usecase.ts";
import { GetProductById } from "./usecases/getById.usecase.ts";
import { ListProducts } from "./usecases/list.usecase.ts";
import { makeProductRoutes } from "./products.routes.ts";

const productQueries = makeProductQueries(db);
const createProduct = new CreateProduct(productQueries);
const getProductById = new GetProductById(productQueries);
const listProducts = new ListProducts(productQueries);

const routes = makeProductRoutes(createProduct, listProducts, getProductById);

export default routes;
```

### 9. Mount Routes

**File**: `src/index.ts`

```ts
import app from './app';
import { envConfig } from "./env.config.ts";
import userRoutes from "./features/users/users.container.ts";
import authRoutes from "./features/auth/auth.container.ts";
import productRoutes from "./features/products/products.container.ts"; // Add this

app.route('/users', userRoutes);
app.route('/auth', authRoutes);
app.route('/products', productRoutes); // Add this

export default {
  port: envConfig.PORT,
  fetch: app.fetch,
};
```

### 10. Tests (Optional but Recommended)

**File**: `src/features/products/tests/Product.spec.ts`

```ts
import { describe, it, expect } from "vitest";
import { Product } from "../Product.ts";

describe("Product Domain Entity", () => {
  it("should fail when creating a product with invalid name", () => {
    const result = Product.create("Ab", "SKU123", 1000, "uuid-123");

    expect(result.ok).toBe(false);
    if (!result.ok) {
      expect(result.error.statusCode).toBe(400);
    }
  });

  it("should create a valid product", () => {
    const result = Product.create("Valid Product", "SKU123", 1000, "uuid-123");

    expect(result.ok).toBe(true);
    if (result.ok) {
      const primitives = result.value.toPrimitives();
      expect(primitives.name).toBe("Valid Product");
      expect(primitives.priceInCents).toBe(1000);
    }
  });
});
```

## Quick Reference

### Command Palette

```bash
# Development
bun run dev                     # Start dev server with hot reload

# Database
bun run db:generate:migration   # Generate migration from schema changes
bun run db:migrate             # Apply pending migrations
bun run db:remove:migration    # Drop a migration

# Testing
bun run test                   # Run all tests
```

### File Structure Template

```
src/features/<feature>/
  <Feature>.ts                  # Domain entity
  <feature>.schemas.ts          # Zod validation schemas
  <feature>.queries.ts          # Database query factory
  <feature>.routes.ts           # HTTP route handlers
  <feature>.container.ts        # Dependency injection wiring
  usecases/
    create.usecase.ts          # Create operation
    getById.usecase.ts         # Get single resource
    list.usecase.ts            # List resources
    update.usecase.ts          # Update operation (optional)
    delete.usecase.ts          # Delete operation (optional)
    errors.ts                  # Feature-specific error helpers
  tests/
    <Feature>.spec.ts          # Domain entity tests
    usecases/
      create.usecase.spec.ts   # Use case tests
```

## Common Patterns

- **UUIDs for IDs**: Always use `uuid('id').primaryKey()`
- **Boolean flags**: Use `is_active` pattern for soft deletes
- **Private constructors**: Domain entities use factory methods
- **Result pattern**: Use cases return `Result<T, ErrorResponse>`
- **Query factories**: Accept `Db` instance, return typed operations
- **Route factories**: Accept use cases, return configured router
- **Middleware order**: Logger → Auth → Routes

## References

For detailed patterns and examples, see:
- `references/feature-scaffold.md` - Copy-paste templates for new features
- `references/common-operations.md` - Update, delete, and other CRUD patterns
