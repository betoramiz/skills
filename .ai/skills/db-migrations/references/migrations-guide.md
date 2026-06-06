# Database Migrations and Seeding Guide

This guide details the conventions and steps to design database tables, generate and apply migrations, and write data seeds using Drizzle ORM and Drizzle Kit.

---

## 1. Schema Conventions

All schemas live under `src/db/schemas/` and are named `<name>.schema.ts`.

### Naming Rules
*   **File name**: lowercase plural or singular depending on the entity, ending with `.schema.ts` (e.g., `users.schema.ts`).
*   **Table Name**: Plural snake_case for the database table (e.g., `pgTable('users', ...)`).
*   **Columns**: CamelCase in TypeScript, snake_case in the database (e.g., `isActive: boolean('is_active')`).
*   **IDs**: Always use UUID for primary keys: `id: uuid('id').primaryKey()`.

### Example Schema (`src/db/schemas/posts.schema.ts`):
```ts
import { pgTable, text, timestamp, uuid } from 'drizzle-orm/pg-core';
import { usersTable } from './users.schema.ts'; // Cross-reference other tables using relative imports

export const postsTable = pgTable('posts', {
  id: uuid('id').primaryKey(),
  title: text('title').notNull(),
  content: text('content').notNull(),
  authorId: uuid('author_id')
    .notNull()
    .references(() => usersTable.id, { onDelete: 'cascade' }),
  createdAt: timestamp('created_at').notNull().defaultNow(),
  updatedAt: timestamp('updated_at').notNull().defaultNow(),
});
```

---

## 2. Registering Schemas

Drizzle Kit and the runtime need an entrypoint to discover all schemas.
Every schema MUST be exported from `src/db/schemas/index.ts`:

```ts
// src/db/schemas/index.ts
export * from './users.schema.ts';
export * from './auth.schema.ts';
export * from './posts.schema.ts';
```

### Relations

If you have relationships between tables, define them in `src/db/schemas/relations.ts`:

```ts
import { relations } from "drizzle-orm";
import { usersTable } from "./users.schema.ts";
import { authTable } from "./auth.schema.ts";

export const usersRelations = relations(usersTable, ({ one }) => ({
  auth: one(authTable, {
    fields: [usersTable.authId],
    references: [authTable.id],
  }),
}));

export const authRelations = relations(authTable, ({ one }) => ({
  user: one(usersTable, {
    fields: [authTable.id],
    references: [usersTable.authId],
  }),
}));
```

---

## 3. Migration Workflow

Once your schema files are defined and registered, follow this local workflow to sync the database:

### Generate Migrations
This command reads the local typescript schema files, compares them to the existing SQL files, and generates a new migration script inside `src/db/drizzle/`:
```bash
bun run db:generate:migration
```
> [!NOTE]
> Review the newly created `.sql` files in `src/db/drizzle/` before applying them to verify they match your intentions.

### Apply Migrations
Apply the generated migration SQL files to the target database:
```bash
bun run db:migrate
```

---

## 4. Database Seeding

To quickly populate local or testing environments, write a seed script. We place seeds under `src/db/seed.ts`.

### Writing a Seed Script
Your seed script should be **idempotent** (safe to run multiple times without duplicating unique records or throwing errors).

```ts
// src/db/seed.ts
import { drizzle } from 'drizzle-orm/postgres-js';
import postgres from 'postgres';
import { envConfig } from '../env.config.ts';
import { usersTable } from './schemas/users.schema.ts';
import { v4 as uuidv4 } from 'uuid';

const sql = postgres(envConfig.DATABASE_URL, { max: 1 });
const db = drizzle(sql);

async function main() {
  console.log('🌱 Seeding database...');

  // 1. Clean existing records (optional/development only)
  // WARNING: Avoid running this in production!
  if (envConfig.NODE_ENV !== 'production') {
    await db.delete(usersTable);
  }

  // 2. Insert seed records
  const seedUsers = [
    {
      id: uuidv4(),
      name: 'John Doe',
      email: 'john@example.com',
      isActive: true,
    },
    {
      id: uuidv4(),
      name: 'Jane Smith',
      email: 'jane@example.com',
      isActive: true,
    },
  ];

  for (const user of seedUsers) {
    await db.insert(usersTable).values(user).onConflictDoNothing({ target: usersTable.email });
  }

  console.log('✅ Seeding complete!');
  await sql.end();
}

main().catch((err) => {
  console.error('❌ Seeding failed:', err);
  process.exit(1);
});
```

### Seeding Execution
Add a seed script runner command to your `package.json` under `"scripts"`:
```json
"db:seed": "bun run src/db/seed.ts"
```
Execute it with:
```bash
bun run db:seed
```
