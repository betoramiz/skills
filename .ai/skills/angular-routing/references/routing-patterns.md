# Routing Patterns

## 1. Route Ownership

- `src/app/app.routes.ts` owns top-level application route entries.
- Every routed feature lives under `src/app/features/<feature-name>/`.
- Feature folders with multiple child routes own their route definition in `src/app/features/<feature-name>/routes.ts`.
- Feature folders own their page component, nested components, facade, service, and models.

## 2. Feature Route

```typescript
import { Routes } from '@angular/router';
import Forms from './forms/forms';
import Buttons from './buttons/buttons';

export const routes: Routes = [
  {
    path: 'forms',
    loadComponent: () => Forms,
  },
  {
    path: 'buttons',
    loadComponent: () => Buttons,
  },
];
```

## 3. Lazy Loading Child Routes

For routes that contain multiple nested or child routes, import the child routes definition file at the top and lazy load it using `loadChildren`:

```typescript
import { Routes } from '@angular/router';
import projectRoutes from './features/project/routes';

export const routes: Routes = [
  {
    path: 'projects',
    loadChildren: () => projectRoutes,
  },
];
```

## 4. Route Target Component

```typescript
@Component({
  selector: 'app-users',
  imports: [PageContent, PageHeader, PageBody],
  templateUrl: './users.html',
  styleUrl: './users.css',
})
export default class Users {}
```

## 5. Anti-Patterns

- Do not use named exports for routable feature pages or child route files.
- Do not create a global routes file under `src/app/features`; feature routing belongs either in `app.routes.ts` for top-level entries or in each feature's own `routes.ts` for child routes.
- Do not use `component:` for new standalone page routes.
- Do not create route paths that differ from feature folder names without a clear product reason.

## 6. Final Checklist

- Page component or nested routes file has a default export.
- Route imports the target from the feature folder.
- Route uses `loadComponent: () => PageClass` or `loadChildren: () => ChildRoutes`.
- Feature path is kebab-case.
- Navigation links, menus, or layout entries are updated when needed.
