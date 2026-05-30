---
name: angular-routing
description: Build or refactor this project's Angular routes. Use when adding feature routes, feature child routes, default-export page components, redirects, lazy loadComponent or loadChildren entries, route file organization, or navigation paths under src/app/app.routes.ts and src/app/features/.
---

# Angular Routing

Use this project's route style instead of generic Angular CLI scaffolding.

## Core Rules

- Keep top-level application routes in `src/app/app.routes.ts`.
- Keep all routed features under `src/app/features/<feature-name>/`.
- For features with child routes, keep the child route definition in `src/app/features/<feature-name>/routes.ts`.
- Route target pages and child route files use default exports.
- Import route targets at the top of the route file and use `loadComponent: () => ComponentClass` for standalone components or `loadChildren: () => ChildRoutes` for nested routes.
- Keep route paths kebab-case and aligned with feature folder names.
- Use relative imports for feature pages and sub-routes inside `features`.
- Add redirects only when the desired default page is explicit.

```typescript
import { Routes } from '@angular/router';
import projectRoutes from './features/project/routes';
import TimesheetComponent from './features/timesheet/timesheet.component';

export const routes: Routes = [
  {
    path: '',
    redirectTo: 'projects',
    pathMatch: 'full',
  },
  {
    path: 'projects',
    loadChildren: () => projectRoutes,
  },
  {
    path: 'timesheet',
    loadComponent: () => TimesheetComponent,
  },
];
```

Read [references/routing-patterns.md](references/routing-patterns.md) before adding a new routed feature or changing layout-level route behavior.
