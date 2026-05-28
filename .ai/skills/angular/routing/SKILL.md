---
name: angular-routing
description: Build or refactor this project's Angular routes. Use when adding dashboard feature routes, layout child routes, default-export page components, redirects, lazy loadComponent entries, route file organization, or navigation paths under src/app/app.routes.ts and src/app/dashboard-features/routes.ts.
---

# Angular Routing

Use this project's route style instead of generic Angular CLI scaffolding.

## Core Rules

- Keep app-level layout routing in `src/app/app.routes.ts`.
- Keep dashboard feature routes in `src/app/dashboard-features/routes.ts`.
- Route target pages use default exports.
- Import route target pages at the top of the route file and use `loadComponent: () => PageClass`.
- Keep route paths kebab-case and aligned with feature folder names.
- Use relative imports for feature pages inside `dashboard-features`.
- Add redirects only when the desired default page is explicit.

```typescript
import { Routes } from '@angular/router';
import Forms from './forms/forms';

export const routes: Routes = [
  {
    path: 'forms',
    loadComponent: () => Forms,
  },
];
```

Read [references/routing-patterns.md](references/routing-patterns.md) before adding a new routed feature or changing layout-level route behavior.
