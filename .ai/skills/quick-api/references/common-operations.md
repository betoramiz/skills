# Common Operations

Quick reference for implementing common CRUD and business operations.

## Update Operation

### Use Case

```ts
import type { ErrorResponse } from "@shared/errors/ErrorTypes.ts";
import { fail, ok, type Result } from "@shared/result.ts";
import type { ProductQueries } from "../products.queries.ts";
import type { UpdateProductCommand } from "../products.schemas.ts";
import { productNotFoundError } from "./errors.ts";

export interface UpdateProductResponse {
  id: string;
  name: string;
  priceInCents: number;
}

export class UpdateProduct {
  constructor(private readonly productQueries: ProductQueries) {}

  async execute(
    productId: string,
    command: UpdateProductCommand
  ): Promise<Result<UpdateProductResponse, ErrorResponse>> {
    const product = await this.productQueries.findById(productId);

    if (product === null) {
      return productNotFoundError();
    }

    // Option 1: If entity has update methods
    const updateResult = product.updateDetails(command.name, command.priceInCents);
    if (!updateResult.ok) {
      return fail(updateResult.error);
    }

    // Option 2: Direct property updates (if entity allows)
    // Note: Only if entity doesn't need validation

    await this.productQueries.save(product);

    const { id, name, priceInCents } = product.toPrimitives();
    return ok({ id, name, priceInCents });
  }
}
```

### Schema

```ts
export const UpdateProductSchema = z.object({
  name: z.string().min(3).optional(),
  priceInCents: z.number().int().min(0).optional(),
});

export type UpdateProductCommand = z.infer<typeof UpdateProductSchema>;
```

### Route

```ts
router.patch('/:id',
  zValidator('param', getProductByIdSchema, validationErrorAsJson),
  zValidator('json', UpdateProductSchema, validationErrorAsJson),
  async (c) => {
    const { id } = c.req.valid('param');
    const body = c.req.valid('json');

    const result = await updateProduct.execute(id, body);
    return respond(c, result);
  }
);
```

### Query Method (if using upsert)

```ts
save: async (product: Product): Promise<void> => {
  const data = product.toPrimitives();

  await database.insert(productsTable)
    .values(data)
    .onConflictDoUpdate({
      target: productsTable.id,
      set: {
        name: data.name,
        priceInCents: data.priceInCents,
      },
    });
},
```

## Delete Operation (Soft Delete)

### Domain Method

Add to entity:

```ts
public deactivate(): void {
  this.props.isActive = false;
}

public activate(): void {
  this.props.isActive = true;
}
```

### Use Case

```ts
import { ok, type Result } from "@shared/result.ts";
import type { ErrorResponse } from "@shared/errors/ErrorTypes.ts";
import type { ProductQueries } from "../products.queries.ts";
import { productNotFoundError } from "./errors.ts";

export class DeactivateProduct {
  constructor(private readonly productQueries: ProductQueries) {}

  async execute(productId: string): Promise<Result<{ id: string; isActive: boolean }, ErrorResponse>> {
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

### Route

```ts
router.delete('/:id',
  zValidator('param', getProductByIdSchema, validationErrorAsJson),
  async (c) => {
    const { id } = c.req.valid('param');

    const result = await deactivateProduct.execute(id);
    return respond(c, result);
  }
);
```

## Delete Operation (Hard Delete)

### Query Method

```ts
deleteById: async (id: string): Promise<boolean> => {
  const result = await database
    .delete(productsTable)
    .where(eq(productsTable.id, id))
    .returning({ id: productsTable.id });

  return result.length > 0;
},
```

### Use Case

```ts
export class DeleteProduct {
  constructor(private readonly productQueries: ProductQueries) {}

  async execute(productId: string): Promise<Result<void, ErrorResponse>> {
    const deleted = await this.productQueries.deleteById(productId);

    if (!deleted) {
      return productNotFoundError();
    }

    return ok(undefined);
  }
}
```

### Route

```ts
router.delete('/:id',
  zValidator('param', getProductByIdSchema, validationErrorAsJson),
  async (c) => {
    const { id } = c.req.valid('param');

    const result = await deleteProduct.execute(id);

    if (!result.ok) {
      return respond(c, result);
    }

    return c.body(null, 204);
  }
);
```

## Search/Filter Operation

### Schema

```ts
export const SearchProductsSchema = z.object({
  q: z.string().min(1).optional(),
  category: z.string().optional(),
  minPrice: z.coerce.number().min(0).optional(),
  maxPrice: z.coerce.number().min(0).optional(),
  isActive: z.coerce.boolean().optional(),
  sortBy: z.enum(['name', 'price', 'createdAt']).default('createdAt'),
  order: z.enum(['asc', 'desc']).default('desc'),
});

export type SearchProductsQuery = z.infer<typeof SearchProductsSchema>;
```

### Query Method

```ts
import { and, or, ilike, gte, lte, eq, desc, asc } from 'drizzle-orm';

search: async (query: SearchProductsQuery) => {
  const conditions = [];

  if (query.q) {
    conditions.push(
      or(
        ilike(productsTable.name, `%${query.q}%`),
        ilike(productsTable.sku, `%${query.q}%`)
      )
    );
  }

  if (query.category) {
    conditions.push(eq(productsTable.category, query.category));
  }

  if (query.minPrice !== undefined) {
    conditions.push(gte(productsTable.priceInCents, query.minPrice));
  }

  if (query.maxPrice !== undefined) {
    conditions.push(lte(productsTable.priceInCents, query.maxPrice));
  }

  if (query.isActive !== undefined) {
    conditions.push(eq(productsTable.isActive, query.isActive));
  }

  const orderColumn = {
    name: productsTable.name,
    price: productsTable.priceInCents,
    createdAt: productsTable.createdAt,
  }[query.sortBy];

  const orderFn = query.order === 'asc' ? asc : desc;

  return database
    .select({
      id: productsTable.id,
      name: productsTable.name,
      sku: productsTable.sku,
      priceInCents: productsTable.priceInCents,
    })
    .from(productsTable)
    .where(conditions.length > 0 ? and(...conditions) : undefined)
    .orderBy(orderFn(orderColumn));
},
```

### Use Case

```ts
export class SearchProducts {
  constructor(private readonly productQueries: ProductQueries) {}

  async execute(query: SearchProductsQuery): Promise<SearchProductsResponse[]> {
    return await this.productQueries.search(query);
  }
}
```

### Route

```ts
router.get('/search',
  zValidator('query', SearchProductsSchema, validationErrorAsJson),
  async (c) => {
    const query = c.req.valid('query');

    const result = await searchProducts.execute(query);
    return c.json(result, 200);
  }
);
```

## Pagination

### Schema

```ts
export const PaginationQuerySchema = z.object({
  page: z.coerce.number().int().min(1).default(1),
  limit: z.coerce.number().int().min(1).max(100).default(20),
});

export type PaginationQuery = z.infer<typeof PaginationQuerySchema>;
```

### Query Method

```ts
import { count } from 'drizzle-orm';

getPaginated: async (query: PaginationQuery) => {
  const offset = (query.page - 1) * query.limit;

  const [data, [{ total }]] = await Promise.all([
    database
      .select({
        id: productsTable.id,
        name: productsTable.name,
        priceInCents: productsTable.priceInCents,
      })
      .from(productsTable)
      .limit(query.limit)
      .offset(offset),
    database
      .select({ total: count() })
      .from(productsTable),
  ]);

  return {
    data,
    pagination: {
      page: query.page,
      limit: query.limit,
      total: total ?? 0,
      totalPages: Math.ceil((total ?? 0) / query.limit),
    },
  };
},
```

### Use Case

```ts
export interface PaginatedProductsResponse {
  data: ProductListItem[];
  pagination: {
    page: number;
    limit: number;
    total: number;
    totalPages: number;
  };
}

export class GetPaginatedProducts {
  constructor(private readonly productQueries: ProductQueries) {}

  async execute(query: PaginationQuery): Promise<PaginatedProductsResponse> {
    return await this.productQueries.getPaginated(query);
  }
}
```

### Route

```ts
router.get('/paginated',
  zValidator('query', PaginationQuerySchema, validationErrorAsJson),
  async (c) => {
    const query = c.req.valid('query');

    const result = await getPaginatedProducts.execute(query);
    return c.json(result, 200);
  }
);
```

## Bulk Operations

### Create Multiple

```ts
export class CreateManyProducts {
  constructor(private readonly productQueries: ProductQueries) {}

  async execute(commands: CreateProductCommand[]): Promise<Result<ProductProps[], ErrorResponse>> {
    const products: Product[] = [];

    for (const command of commands) {
      const productResult = Product.create(
        command.name,
        command.sku,
        command.priceInCents,
        uuidv4()
      );

      if (!productResult.ok) {
        return fail(productResult.error);
      }

      products.push(productResult.value);
    }

    await this.productQueries.saveMany(products);

    return ok(products.map(p => p.toPrimitives()));
  }
}
```

### Query Method

```ts
saveMany: async (products: Product[]): Promise<void> => {
  const values = products.map(p => p.toPrimitives());

  await database.insert(productsTable).values(values);
},
```

## Aggregate Queries

### Count by Status

```ts
import { count, eq } from 'drizzle-orm';

getStatistics: async () => {
  const [activeCount, inactiveCount, totalRevenue] = await Promise.all([
    database
      .select({ count: count() })
      .from(productsTable)
      .where(eq(productsTable.isActive, true)),
    database
      .select({ count: count() })
      .from(productsTable)
      .where(eq(productsTable.isActive, false)),
    database
      .select({ sum: sql<number>`SUM(${productsTable.priceInCents})` })
      .from(productsTable)
      .where(eq(productsTable.isActive, true)),
  ]);

  return {
    activeProducts: activeCount[0]?.count ?? 0,
    inactiveProducts: inactiveCount[0]?.count ?? 0,
    totalInventoryValue: totalRevenue[0]?.sum ?? 0,
  };
},
```

## Relationships

### One-to-Many Query

```ts
import { eq } from 'drizzle-orm';

getProductWithReviews: async (productId: string) => {
  const rows = await database
    .select({
      product: productsTable,
      review: reviewsTable,
    })
    .from(productsTable)
    .leftJoin(reviewsTable, eq(reviewsTable.productId, productsTable.id))
    .where(eq(productsTable.id, productId));

  if (rows.length === 0) return null;

  const product = rows[0].product;
  const reviews = rows
    .filter(row => row.review !== null)
    .map(row => row.review!);

  return {
    ...product,
    reviews,
  };
},
```

## Best Practices

1. **Soft delete by default** - use `isActive` flag
2. **Validate in domain** - entities validate business rules
3. **Partial updates** - use `.optional()` on update schemas
4. **Pagination for lists** - when data can grow large
5. **Batch operations** - validate all before persisting any
6. **Filter in queries** - keep business logic out of routes
7. **Return counts** - include total count for paginated results
8. **Use transactions** - for operations spanning multiple tables
