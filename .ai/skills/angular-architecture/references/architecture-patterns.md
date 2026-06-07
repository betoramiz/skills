# Architecture & Modular Scaffolding Reference

This document details the architectural conventions, file naming standards, path aliases, and structural flow patterns enforced across the Angular codebase.

---

## 1. Feature Module Scaffolding Blueprint

When creating a new domain feature under `src/app/features/`, follow this exact folder structure:

```
src/app/features/<feature-name>/
├── components/                      # Private, domain-specific components
│   └── <inner-component>/
│       ├── <inner-component>.ts
│       ├── <inner-component>.html
│       └── <inner-component>.css
├── models/                          # Feature data definitions
│   ├── <feature-name>-models.ts     # TypeScript interfaces and types
│   └── mappers.ts                   # DTO mapping conversion functions
├── resolvers/                       # Route data preloaders
│   └── <action>.resolver.ts         # e.g. get-all-projects.resolver.ts
├── <feature-name>-service.ts        # REST service extending BaseService
├── <feature-name>-facade.ts         # Controller managing signals and subscriptions
├── routes.ts                        # Feature child routes (default export)
├── <feature-name>.ts                # Main page entry component (default export)
├── <feature-name>.html
└── <feature-name>.css
```

---

## 2. Shared Core Module (`src/app/shared/`)

The shared module is a central directory for reusable utilities, UI layouts, types, and services. It is divided into logical subfolders:

- `components/layout/`: Common wrappers and page layouts, such as `page-content`, `page-header`, `page-body`.
- `components/dialgos/`: Centralized modal widgets (e.g. `confirmation-component.ts`, `error-component.ts`, `yes-no-component.ts`). **Notice the folder name spelling: "dialgos"**.
- `services/`: Infrastructure utilities like `BaseService` (REST core class), `BaseCrudFacade` (reusable facade base class), and `ModalService` (dialog trigger coordinator).
- `models/`: App-wide structures, such as `Pagination` and common type declarations.

---

## 3. Path Mapping / Aliases

The project utilizes TypeScript path mapping defined in `tsconfig.json`. Use the four granular aliases — never use `@shared/` (no sub-scope).

```json
"paths": {
  "@shared-component/*": ["./src/app/shared/components/*"],
  "@shared-directives/*": ["./src/app/shared/directives/*"],
  "@shared-model/*":      ["./src/app/shared/models/*"],
  "@shared-services/*":   ["./src/app/shared/services/*"]
}
```

> **Note**: `@shared-component` and `@shared-model` are **singular**; `@shared-services` and `@shared-directives` are **plural**. Never write `@shared-components` (plural) or `@shared/` (no sub-scope).

### Import Rules:
- **Shared items**: Always use the appropriate granular alias.
- **Feature items**: Use relative imports (`./components/...`, `./models/...`) when importing files *within the same feature module*.

### Concrete examples:
```typescript
import { PageBody } from '@shared-component/layout';
import { ModalService } from '@shared-services/modal-service';
import { Pagination } from '@shared-model/pagination';
import { ActionStatus } from '@shared-model/types';
```

---

## 4. UI Layout & Styles Stack

The application blends **Tailwind CSS v4** and **Angular Material 3 (M3)** using a unified styles system:

1. **Global Stylesheet**: `src/styles.css` imports Tailwind directives using `@import "tailwindcss";`.
2. **Material Custom Theme**: `src/themes/custom-theme.scss` loads `@angular/material` as `mat`, sets global typography to Roboto, and handles style overrides using dedicated design tokens and maps.
3. **Template Grid & Flexbox**: Components must use Tailwind grid and flex utilities (`grid`, `flex`, `gap-4`, `p-6`) for fast, clean, responsive positioning.

---

## 5. Resolver Placement

Resolvers preload data before a route activates, preventing blank page flashes.

- **Location**: `features/<feature>/resolvers/<action>.resolver.ts`
  Examples: `get-all-projects.resolver.ts`, `get-by-id.resolver.ts`
- **Rule**: Resolvers call the feature service, **not** the facade.
- **Route attachment**: `resolve: { keyName: resolverFn }`
- **Data access in component constructor** (synchronous):
  ```typescript
  const data = inject(ActivatedRoute).snapshot.data['keyName'] as T;
  this.facade.setAll(data ?? []);
  ```
- See `angular-routing` skill for the `ResolveFn<T>` function signature and anti-patterns.

---

## 6. Related Skill Boundaries

- Use `angular-component` for standalone component APIs, signal inputs/outputs, and template control flow.
- Use `angular-form-component` for typed reactive form UI and validation behavior.
- Use `angular-facade` for feature-local state orchestration.
- Use `angular-service` for REST wrappers and endpoint construction.
- Use `angular-routing` for route entries, navigation structure, and resolver functions.
- Use `angular-testing` for focused verification of changed behavior.

---

## 7. Architecture Anti-Patterns

- Do not create a feature without deciding whether it needs a facade and service.
- Do not place domain-only UI in `src/app/shared`.
- Do not duplicate DTO-to-form mapping inside components.
- Do not introduce a new path alias unless repeated imports justify it.
- Do not fix the `dialgos` typo opportunistically in unrelated work.
- Do not use `@shared/` (no sub-scope) — always use the four granular `@shared-*` aliases.

## 8. Feature Checklist

- Folder lives under `src/app/features/<feature-name>/`.
- Routable page files use `<feature-name>.ts/html/css` with `export default class`.
- Nested UI lives under `components/`.
- Models and mappers live under `models/`.
- Resolvers live under `resolvers/` and are registered in `routes.ts`.
- Route entry and navigation entry are updated when the page should be reachable.
- Shared imports use the four granular `@shared-*` aliases.
- Relevant tests are added or updated for changed behavior.
