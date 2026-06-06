# Detailed Use Case Examples

These examples use a hypothetical `products` feature while matching the current `users` patterns.

## Example 1: Create Product

### Request Schema

`src/features/products/products.schemas.ts`:

```ts
import { z } from "zod";

export const CreateProductSchema = z.object({
  name: z.string().min(3),
  sku: z.string().min(3),
  priceInCents: z.number().int().min(0),
});

export type CreateProductCommand = z.infer<typeof CreateProductSchema>;
```

### Domain Entity

`src/features/products/Product.ts`:

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

### Use Case

`src/features/products/usecases/create.usecase.ts`:

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

### Route Handler

```ts
router.post('/', zValidator('json', CreateProductSchema, validationErrorAsJson), async (c) => {
  const body = c.req.valid("json");

  const result = await createProduct.execute(body);
  return respond(c, result, 201);
});
```

### Why This Is Correct

- The route validates request shape and delegates behavior.
- The use case creates the ID, calls the domain factory, persists the entity, and returns primitives.
- The domain entity owns business invariants.
- Duplicate SKU/database uniqueness problems can bubble to `app.onError`, where Postgres unique violations map to conflict responses.

## Example 2: Get Product By Id

### Request Schema

```ts
export const getProductByIdSchema = z.object({
  id: z.uuid(),
});

export type GetProductByIdCommand = z.infer<typeof getProductByIdSchema>;
```

### Error Helper

`src/features/products/usecases/errors.ts`:

```ts
import { AppError } from "@shared/errors/AppError.ts";
import { fail } from "@shared/result.ts";

export const productNotFoundError = (message?: string) =>
  fail(AppError.NotFound(message ?? 'Product Not Found'));
```

### Query Method

```ts
findById: async (id: string): Promise<Product | null> => {
  const [row] = await database.select().from(productsTable).where(eq(productsTable.id, id));
  if (!row) return null;

  return Product.reconstitute(row);
}
```

### Use Case

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

### Route Handler

```ts
router.get('/:id', zValidator('param', getProductByIdSchema, validationErrorAsJson), async (c) => {
  const { id } = c.req.valid('param');

  const result = await getProductById.execute(id);
  return respond(c, result);
});
```

### Why This Is Correct

- The route validates the UUID before calling the use case.
- The use case maps missing persistence rows into an expected `Result` failure.
- The response hides internal fields such as `isActive` unless they are intentionally part of the API.

## Example 3: Rename Product

This example shows an update operation with a domain method.

### Schema

```ts
export const RenameProductSchema = z.object({
  name: z.string().min(3),
});

export type RenameProductCommand = z.infer<typeof RenameProductSchema>;
```

### Domain Method

Add this to `Product`:

```ts
public rename(name: string): Result<void, ErrorResponse> {
  const trimmedName = name.trim();

  if (trimmedName.length < 3) {
    return fail(AppError.BadRequest("Product name must have at least 3 characters."));
  }

  this.props.name = trimmedName;
  return ok(undefined);
}
```

### Use Case

```ts
import type { ErrorResponse } from "@shared/errors/ErrorTypes.ts";
import { fail, ok, type Result } from "@shared/result.ts";
import type { ProductQueries } from "../products.queries.ts";
import type { RenameProductCommand } from "../products.schemas.ts";
import { productNotFoundError } from "./errors.ts";

export interface RenameProductResponse {
  id: string;
  name: string;
}

export class RenameProduct {
  constructor(private readonly productQueries: ProductQueries) {}

  async execute(
    productId: string,
    command: RenameProductCommand
  ): Promise<Result<RenameProductResponse, ErrorResponse>> {
    const product = await this.productQueries.findById(productId);
    if (product === null) {
      return productNotFoundError();
    }

    const renameResult = product.rename(command.name);
    if (!renameResult.ok) {
      return fail(renameResult.error);
    }

    await this.productQueries.save(product);

    const { id, name } = product.toPrimitives();
    return ok({ id, name });
  }
}
```

### Route Handler

```ts
router.patch(
  '/:id/rename',
  zValidator('param', getProductByIdSchema, validationErrorAsJson),
  zValidator('json', RenameProductSchema, validationErrorAsJson),
  async (c) => {
    const { id } = c.req.valid('param');
    const body = c.req.valid('json');

    const result = await renameProduct.execute(id, body);
    return respond(c, result);
  }
);
```

### Why This Is Correct

- The route parses HTTP input and stops there.
- The use case loads the entity, asks it to perform a business operation, saves the result, and returns a DTO.
- The entity protects its invariant in both create and rename paths.

## Example 4: Deactivate Product

### Domain Method

```ts
public deactivate(): void {
  this.props.isActive = false;
}
```

### Use Case

```ts
import type { ErrorResponse } from "@shared/errors/ErrorTypes.ts";
import { ok, type Result } from "@shared/result.ts";
import type { ProductQueries } from "../products.queries.ts";
import { productNotFoundError } from "./errors.ts";

export interface DeactivateProductResponse {
  id: string;
  isActive: boolean;
}

export class DeactivateProduct {
  constructor(private readonly productQueries: ProductQueries) {}

  async execute(productId: string): Promise<Result<DeactivateProductResponse, ErrorResponse>> {
    const product = await this.productQueries.findById(productId);
    if (product === null) {
      return productNotFoundError();
    }

    product.deactivate();
    await this.productQueries.save(product);

    const { id, isActive } = product.toPrimitives();
    return ok({ id, isActive });
  }
}
```

### Why This Is Correct

- The use case treats missing products as an expected application failure.
- The state change is expressed on the entity.
- Persistence remains behind the query factory.

## Example 5: List Active Products

### Response Type and Use Case

```ts
import type { ProductQueries } from "../products.queries.ts";

export interface ListActiveProductsResponse {
  id: string;
  name: string;
  sku: string;
  priceInCents: number;
}

export class ListActiveProducts {
  constructor(private readonly productQueries: ProductQueries) {}

  async execute(): Promise<ListActiveProductsResponse[]> {
    let products: ListActiveProductsResponse[];

    products = await this.productQueries.getActive();
    return products;
  }
}
```

### Query Method

```ts
getActive: async (): Promise<ListActiveProductsResponse[]> => {
  return database.select({
    id: productsTable.id,
    name: productsTable.name,
    sku: productsTable.sku,
    priceInCents: productsTable.priceInCents,
  })
    .from(productsTable)
    .where(eq(productsTable.isActive, true));
}
```

### Route Handler

```ts
router.get('/active', async (c) => {
  const result = await listActiveProducts.execute();
  return c.json(result, 200);
});
```

### Why This Is Correct

- The list use case mirrors the existing users list style.
- The query returns a response projection directly.
- The route uses direct JSON because the use case does not return `Result`.

## Common Mistakes To Avoid

- Do not import `Context` from Hono in a use case.
- Do not return `c.json(...)` from a use case.
- Do not run Drizzle queries inside route handlers.
- Do not bypass entity factories for new domain objects.
- Do not throw `AppError` for expected conditions; return `fail(AppError.*(...))`.
- Do not expose full primitives if the response should hide fields.
- Do not forget to wire the new use case in the feature container.
