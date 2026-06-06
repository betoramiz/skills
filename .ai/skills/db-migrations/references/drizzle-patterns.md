# Advanced Drizzle ORM Patterns

This guide outlines advanced query patterns, relational joins, complex filters, and clean transaction patterns within a modular TypeScript backend.

---

## 1. Complex Query Filtering

Import the query builders from `drizzle-orm` and `drizzle-orm/pg-core`. Always perform filter logic within the Query Factory layer (`src/features/<feature>/<feature>.queries.ts`), never in use cases or routes.

### Logical Operators and Text Search
```ts
import { eq, and, or, ilike, inArray, desc } from 'drizzle-orm';
import { usersTable } from '@db/schemas/index.ts';

// Query Factory method example
async function searchUsers(
  searchTerm?: string,
  roles?: string[]
) {
  const conditions = [];

  if (searchTerm) {
    conditions.push(
      or(
        ilike(usersTable.name, `%${searchTerm}%`),
        ilike(usersTable.email, `%${searchTerm}%`)
      )
    );
  }

  if (roles && roles.length > 0) {
    conditions.push(inArray(usersTable.role, roles));
  }

  return await database
    .select()
    .from(usersTable)
    .where(and(...conditions))
    .orderBy(desc(usersTable.createdAt));
}
```

---

## 2. Relational Joins and Mapping

When querying related tables, Drizzle returns a flat array of objects (one per joined row). You must map this flat list into a hierarchical structure or your domain aggregates.

### Example: One-to-Many Join (Posts with Comments)

```ts
import { eq } from 'drizzle-orm';
import { postsTable, commentsTable } from '@db/schemas/index.ts';

export const makePostQueries = (database: Db) => ({
  findPostWithComments: async (postId: string) => {
    const rows = await database
      .select({
        post: postsTable,
        comment: commentsTable,
      })
      .from(postsTable)
      .leftJoin(commentsTable, eq(commentsTable.postId, postsTable.id))
      .where(eq(postsTable.id, postId));

    if (rows.length === 0) return null;

    const postRow = rows[0].post;
    
    // Aggregate comments from rows
    const comments = rows
      .filter((row) => row.comment !== null)
      .map((row) => ({
        id: row.comment!.id,
        content: row.comment!.content,
        createdAt: row.comment!.createdAt,
      }));

    return {
      id: postRow.id,
      title: postRow.title,
      content: postRow.content,
      comments,
    };
  }
});
```

---

## 3. Clean Transaction Handling

To maintain **Clean Architecture**, use cases must remain decoupled from specific database technologies. However, business transactions often span multiple queries that must be atomic. 

Drizzle handles transactions via `db.transaction(async (tx) => { ... })`. Both the main database connection (`db`) and the transaction client (`tx`) share the same type signature in Drizzle (`PgDatabase` / generic database runner type). We call this type `Db`.

### Pattern: Injected Transaction Context

To run queries atomically inside a use case without introducing `db` dependencies:
1. Define a transaction-aware unit of work or allow passing a `tx` client to queries.
2. In this architecture, query factories receive the database runner `Db`. We can instantiate query factories dynamically using a transaction instance.

#### Step A: Enable transactional Query Factory construction
Query factories must accept the `Db` client. When instantiating inside a transaction, we pass the transaction client (`tx`) to a new instance of the query factory.

```ts
// src/features/orders/orders.queries.ts
import type { Db } from '@db/connection.ts';
import { ordersTable } from '@db/schemas/index.ts';

export const makeOrderQueries = (database: Db) => ({
  save: async (order: Order): Promise<void> => {
    await database.insert(ordersTable).values(order.toPrimitives());
  },
  // ... other queries
});

export type OrderQueries = ReturnType<typeof makeOrderQueries>;
```

#### Step B: Use Case Transaction Execution
Create a transaction manager inside `src/db/` or inject the database wrapper to orchestrate the transaction. The Use Case initiates the transaction, creates transient queries using the transaction client `tx`, and executes the operations.

```ts
// src/db/connection.ts
import { drizzle } from 'drizzle-orm/postgres-js';
import postgres from 'postgres';

export const client = postgres(envConfig.DATABASE_URL);
export const db = drizzle(client);

// The type representing either db or tx in Drizzle
export type Db = typeof db;
```

```ts
// src/features/orders/usecases/checkout.usecase.ts
import { ok, fail, type Result } from '@shared/result.ts';
import { AppError } from '@shared/errors/AppError.ts';
import type { Db } from '@db/connection.ts';
import { makeOrderQueries } from '../orders.queries.ts';
import { makeInventoryQueries } from '../../inventory/inventory.queries.ts';

export class CheckoutUseCase {
  // Inject the core db client to manage the transaction boundary
  constructor(private readonly rawDb: Db) {}

  async execute(command: CheckoutCommand): Promise<Result<CheckoutResponse, ErrorResponse>> {
    try {
      // Execute the database transaction
      const result = await this.rawDb.transaction(async (tx) => {
        // 1. Create transaction-scoped query factories
        const orderQueries = makeOrderQueries(tx);
        const inventoryQueries = makeInventoryQueries(tx);

        // 2. Perform business checks (e.g. check stock)
        const stock = await inventoryQueries.checkStock(command.itemId);
        if (stock < command.quantity) {
          return fail(AppError.BadRequest('Insufficient inventory stock.'));
        }

        // 3. Deduct stock
        await inventoryQueries.deductStock(command.itemId, command.quantity);

        // 4. Create and save order
        const orderId = crypto.randomUUID();
        await orderQueries.save({
          id: orderId,
          itemId: command.itemId,
          quantity: command.quantity,
          status: 'PENDING',
        });

        return ok({ orderId });
      });

      return result;
    } catch (error) {
      // Unexpected database failures bubble up to global Hono error handlers
      throw error;
    }
  }
}
```

> [!TIP]
> Always return `fail(AppError.*)` for business logic failures inside transactions to prevent unhandled database exceptions. Return `fail` values inside the transaction block; Drizzle will commit or you can throw an exception explicitly to force rollbacks if your business logic dictates a rollback.
>
> If you return a `Failure` Result, Drizzle does **not** roll back automatically unless an error is thrown. If you need a rollback on a failed business Result:
> ```ts
> await db.transaction(async (tx) => {
>   const result = await performAction(tx);
>   if (!result.ok) {
>     tx.rollback(); // Explicitly rolls back the transaction in Drizzle
>     return fail(result.error);
>   }
>   return ok(result.value);
> });
> ```
