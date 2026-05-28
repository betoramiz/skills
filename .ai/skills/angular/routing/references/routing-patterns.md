# Routing Patterns

## 1. Route Ownership

- `src/app/app.routes.ts` owns top-level layout selection.
- `src/app/dashboard-features/routes.ts` owns pages rendered inside the dashboard layout.
- Feature folders own their page component, nested components, facade, service, and models.

## 2. Dashboard Feature Route

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

## 3. Route Target Component

```typescript
@Component({
  selector: 'app-users',
  imports: [PageContent, PageHeader, PageBody],
  templateUrl: './users.html',
  styleUrl: './users.css',
})
export default class Users {}
```

## 4. Anti-Patterns

- Do not use named exports for routable feature pages.
- Do not put dashboard feature routes directly in `app.routes.ts`.
- Do not use `component:` for new standalone page routes.
- Do not create route paths that differ from feature folder names without a clear product reason.

## 5. Final Checklist

- Page component has a default export.
- Route imports the page from the feature folder.
- Route uses `loadComponent: () => PageClass`.
- Feature path is kebab-case.
- Navigation links, menus, or layout entries are updated when needed.
