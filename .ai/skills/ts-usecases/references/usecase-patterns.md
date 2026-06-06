# Use Case Patterns

Use cases in this project live under `src/features/<feature>/usecases/`. They represent application behavior and coordinate domain entities with persistence queries.

## Common Shape

```ts
export class SomeUseCase {
  constructor(private readonly featureQueries: FeatureQueries) {}

  async execute(command: SomeCommand): Promise<Result<SomeResponse, ErrorResponse>> {
    // orchestration
  }
}
```

Use cases should:

- Receive dependencies through the constructor.
- Expose one `execute` method.
- Use Zod-inferred command types for request-driven operations.
- Return `Result<T, ErrorResponse>` for expected success/failure flows.
- Return plain data only when the existing route pattern intentionally does so.
- Avoid Hono imports, Hono contexts, and HTTP status codes.
- Avoid direct Drizzle operations; call query factory methods instead.

## Create Pattern

Use this pattern when creating a new aggregate or entity.

```ts
import { v4 as uuidv4 } from "uuid";
import { Entity, type EntityProps } from "../Entity.ts";
import type { FeatureQueries } from "../feature.queries.ts";
import type { CreateEntityCommand } from "../feature.schemas.ts";
import type { ErrorResponse } from "@shared/errors/ErrorTypes.ts";
import { fail, ok, type Result } from "@shared/result.ts";

export class CreateEntity {
  constructor(private readonly featureQueries: FeatureQueries) {}

  async execute(command: CreateEntityCommand): Promise<Result<EntityProps, ErrorResponse>> {
    const entityResult = Entity.create(command.name, uuidv4());

    if (!entityResult.ok) {
      return fail(entityResult.error);
    }

    const entity = entityResult.value;
    await this.featureQueries.save(entity);

    return ok(entity.toPrimitives());
  }
}
```

Guidelines:

- Generate IDs in the use case unless the domain has a stronger ID policy.
- Put invariant checks in `Entity.create`.
- Return the domain validation failure unchanged.
- Persist only after the entity is valid.
- Return primitives or a response DTO, not the class instance.

## Get By Id Pattern

Use this pattern when loading one entity by identifier.

```ts
import { ok, type Result } from "@shared/result.ts";
import type { ErrorResponse } from "@shared/errors/ErrorTypes.ts";
import type { FeatureQueries } from "../feature.queries.ts";
import type { Entity } from "../Entity.ts";
import { entityNotFoundError } from "./errors.ts";

export interface GetEntityByIdResponse {
  id: string;
  name: string;
}

export class GetEntityById {
  constructor(private readonly featureQueries: FeatureQueries) {}

  async execute(entityId: string): Promise<Result<GetEntityByIdResponse, ErrorResponse>> {
    let entity: Entity | null;

    entity = await this.featureQueries.findById(entityId);
    if (entity === null) {
      return entityNotFoundError();
    }

    const { id, name } = entity.toPrimitives();
    return ok({ id, name });
  }
}
```

Guidelines:

- Query methods should return `Entity | null` for not-found reads.
- Map `null` to a feature-specific not-found error helper.
- Return only fields that belong in the response.

## List Pattern

The current users feature returns a plain list:

```ts
export interface GetListResponse {
  id: string;
  name: string;
  email: string;
}

export class GetList {
  constructor(private readonly userQueries: UserQueries) {}

  async execute(): Promise<GetListResponse[]> {
    let users: GetListResponse[];

    users = await this.userQueries.getAll();
    return users;
  }
}
```

Use this shape when matching existing list routes that respond directly with `c.json(result, status)`.

Prefer a `Result` list when the route will use `respond` consistently:

```ts
export class ListEntities {
  constructor(private readonly featureQueries: FeatureQueries) {}

  async execute(): Promise<Result<ListEntityResponse[], ErrorResponse>> {
    const entities = await this.featureQueries.getAll();
    return ok(entities);
  }
}
```

Do not mix both shapes in the same route handler.

## Update Pattern

Use this pattern for changes to an existing entity.

```ts
export interface UpdateEntityResponse {
  id: string;
  name: string;
  isActive: boolean;
}

export class UpdateEntity {
  constructor(private readonly featureQueries: FeatureQueries) {}

  async execute(
    entityId: string,
    command: UpdateEntityCommand
  ): Promise<Result<UpdateEntityResponse, ErrorResponse>> {
    const entity = await this.featureQueries.findById(entityId);
    if (entity === null) {
      return entityNotFoundError();
    }

    const updateResult = entity.rename(command.name);
    if (!updateResult.ok) {
      return fail(updateResult.error);
    }

    await this.featureQueries.save(entity);

    const { id, name, isActive } = entity.toPrimitives();
    return ok({ id, name, isActive });
  }
}
```

Guidelines:

- Add domain methods to the entity for invariant-preserving changes.
- Return `fail(updateResult.error)` when a domain method returns `Result`.
- Save after the domain change succeeds.
- Keep partial update merge rules in the use case or a domain method, not in the route.

## Delete or Deactivate Pattern

Prefer soft state changes when the entity already has `isActive`.

```ts
export class DeactivateEntity {
  constructor(private readonly featureQueries: FeatureQueries) {}

  async execute(entityId: string): Promise<Result<{ id: string; isActive: boolean }, ErrorResponse>> {
    const entity = await this.featureQueries.findById(entityId);
    if (entity === null) {
      return entityNotFoundError();
    }

    entity.deactivate();
    await this.featureQueries.save(entity);

    const { id, isActive } = entity.toPrimitives();
    return ok({ id, isActive });
  }
}
```

Only hard delete when the product behavior requires it. If hard deleting, add a clear query method such as `deleteById` and decide whether not-found is detected before or after deletion.

## Error Helper Pattern

Feature expected errors live in `usecases/errors.ts`.

```ts
import { AppError } from "@shared/errors/AppError.ts";
import { fail } from "@shared/result.ts";

export const entityNotFoundError = (message?: string) =>
  fail(AppError.NotFound(message ?? 'Entity Not Found'));
```

Use helpers when:

- Multiple use cases need the same error.
- A message should be consistent across the feature.
- The error represents expected application flow.

Do not use helpers for unexpected infrastructure failures. Let those throw and rely on the global error handler.

## Query Method Pairing

Every use case dependency must exist in the feature query factory.

For a use case that calls:

```ts
await this.productQueries.findBySku(command.sku);
await this.productQueries.save(product);
```

`products.queries.ts` must return:

```ts
export const makeProductQueries = (database: Db) => ({
  findBySku: async (sku: string): Promise<Product | null> => {
    const [row] = await database.select().from(productsTable).where(eq(productsTable.sku, sku));
    if (!row) return null;

    return Product.reconstitute(row);
  },

  save: async (product: Product): Promise<void> => {
    const data = product.toPrimitives();
    await database.insert(productsTable).values(data);
  },
});
```

Because `FeatureQueries` is `ReturnType<typeof makeFeatureQueries>`, adding the method to the factory updates the injected type automatically.

## Route Pairing

Routes should pass validated data into use cases:

```ts
router.patch(
  '/:id',
  zValidator('param', getByIdSchema, validationErrorAsJson),
  zValidator('json', UpdateEntitySchema, validationErrorAsJson),
  async (c) => {
    const { id } = c.req.valid('param');
    const body = c.req.valid('json');

    const result = await updateEntity.execute(id, body);
    return respond(c, result);
  }
);
```

Use cases should not parse route params themselves.
