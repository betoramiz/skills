# Detailed Feature Module Example

This example adds a complete `products` feature using the same architecture as the current `users` feature.

## Target Structure

```text
src/db/schemas/products.schema.ts
src/features/products/Product.ts
src/features/products/products.schemas.ts
src/features/products/products.queries.ts
src/features/products/products.routes.ts
src/features/products/products.container.ts
src/features/products/usecases/create.usecase.ts
src/features/products/usecases/getById.usecase.ts
src/features/products/usecases/list.usecase.ts
src/features/products/usecases/errors.ts
```

Also update:

```text
src/db/schemas/index.ts
src/index.ts
```

## Database Schema

Create `src/db/schemas/products.schema.ts`:

```ts
import { boolean, integer, pgTable, text, uuid } from 'drizzle-orm/pg-core';

export const productsTable = pgTable('products', {
  id: uuid('id').primaryKey(),
  name: text('name').notNull(),
  sku: text('sku').notNull().unique(),
  priceInCents: integer('price_in_cents').notNull(),
  isActive: boolean('is_active').notNull().default(true),
});
```

Update `src/db/schemas/index.ts`:

```ts
export * from './users.schema.ts'
export * from './products.schema.ts'
```

## Domain Entity

Create `src/features/products/Product.ts`:

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

    if (sku.trim().length < 3) {
      return fail(AppError.BadRequest("Product SKU must have at least 3 characters."));
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

Why this matches the project:

- The constructor is private.
- `create` validates domain invariants and returns `Result`.
- `reconstitute` rebuilds an entity from persistence.
- `toPrimitives` returns plain data for persistence or responses.

## Request Schemas

Create `src/features/products/products.schemas.ts`:

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

export type GetProductByIdCommand = z.infer<typeof getProductByIdSchema>;
```

Keep request validation here. Do not duplicate these checks in the route handler.

## Query Factory

Create `src/features/products/products.queries.ts`:

```ts
import { eq } from "drizzle-orm";
import { type Db } from "@db/connection.ts";
import { productsTable } from "@db/schemas";
import { Product } from "./Product.ts";
import type { ListProductsResponse } from "./usecases/list.usecase.ts";

export const makeProductQueries = (database: Db) => ({
  getAll: async (): Promise<ListProductsResponse[]> => {
    return database.select({
      id: productsTable.id,
      name: productsTable.name,
      sku: productsTable.sku,
      priceInCents: productsTable.priceInCents,
    })
      .from(productsTable);
  },

  save: async (product: Product): Promise<void> => {
    const data = product.toPrimitives();

    await database.insert(productsTable)
      .values(data)
      .onConflictDoUpdate({
        target: productsTable.id,
        set: {
          name: data.name,
          sku: data.sku,
          priceInCents: data.priceInCents,
          isActive: data.isActive,
        },
      });
  },

  findById: async (id: string): Promise<Product | null> => {
    const [row] = await database.select().from(productsTable).where(eq(productsTable.id, id));
    if (!row) return null;

    return Product.reconstitute(row);
  },
});

export type ProductQueries = ReturnType<typeof makeProductQueries>;
```

Why this matches the project:

- The database is injected, not imported directly.
- Drizzle code stays in the query factory.
- `findById` returns a domain entity because the use case may need behavior.
- `getAll` returns a projection because a list response does not need full domain behavior.

## Use Case Errors

Create `src/features/products/usecases/errors.ts`:

```ts
import { AppError } from "@shared/errors/AppError.ts";
import { fail } from "@shared/result.ts";

export const productNotFoundError = (message?: string) =>
  fail(AppError.NotFound(message ?? 'Product Not Found'));
```

Use helpers for expected feature errors that may be reused by multiple use cases.

## Create Use Case

Create `src/features/products/usecases/create.usecase.ts`:

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

## Get By Id Use Case

Create `src/features/products/usecases/getById.usecase.ts`:

```ts
import { ok, type Result } from "@shared/result.ts";
import type { ErrorResponse } from "@shared/errors/ErrorTypes.ts";
import type { ProductQueries } from "../products.queries.ts";
import type { Product } from "../Product.ts";
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
    let product: Product | null;

    product = await this.productQueries.findById(productId);
    if (product === null) {
      return productNotFoundError();
    }

    const { id, name, sku, priceInCents } = product.toPrimitives();
    return ok({ id, name, sku, priceInCents });
  }
}
```

## List Use Case

Create `src/features/products/usecases/list.usecase.ts`:

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
    let products: ListProductsResponse[];

    products = await this.productQueries.getAll();
    return products;
  }
}
```

This mirrors the current `GetList` pattern in users, which returns a plain list rather than `Result`.

## Routes

Create `src/features/products/products.routes.ts`:

```ts
import { Hono } from "hono";
import { zValidator } from "@hono/zod-validator";
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

Notes:

- Use `respond` for `Result` values.
- Use `validationErrorAsJson` for Zod validation failures.
- Keep route handlers thin.

## Container

Create `src/features/products/products.container.ts`:

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

## Mount Routes

Update `src/index.ts`:

```ts
import app from './app';
import { envConfig } from "./env.config.ts";
import userRoutes from "./features/users/users.container.ts";
import productRoutes from "./features/products/products.container.ts";

app.route('/users', userRoutes)
app.route('/products', productRoutes)

export default {
  port: envConfig.PORT,
  fetch: app.fetch,
};
```

## Validation Checklist

- The new feature has a container, routes, schemas, queries, entity, and use cases.
- The new database schema is exported from `src/db/schemas/index.ts`.
- The route group is mounted in `src/index.ts`.
- All local imports use `.ts` extensions unless importing from an index export that already follows the local convention.
- Expected errors use `Result` and `AppError`.
- Query factories accept `Db` and do not import `db` directly.
- Use cases do not import Hono.
