# Routing Patterns

## 1. Route Ownership

- `src/app/app.routes.ts` owns top-level application route entries.
- Every routed feature lives under `src/app/features/<feature-name>/`.
- Feature folders with multiple child routes own their route definition in `src/app/features/<feature-name>/routes.ts`.
- Feature folders own their page component, nested components, facade, service, models, and resolvers.

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

## 5. Resolver Pattern

Use resolvers to preload data before the route activates, so the component always has data available on init.

### Resolver Function

- Type: `ResolveFn<T>` from `@angular/router`
- Use `inject()` inside the function body — not constructor injection
- Return the service observable directly (no `.subscribe()`, no `runQuery`)
- Named exports only — never default export a resolver

```typescript
// features/project/resolvers/get-all-projects.resolver.ts
import { ResolveFn } from '@angular/router';
import { inject } from '@angular/core';
import { ProjectService } from '../project.service';
import { ProjectListItemDto } from '../models/project-model';

export const getAllProjectsResolver: ResolveFn<ProjectListItemDto[]> = (route, state) => {
  return inject(ProjectService).getListOf<ProjectListItemDto>('');
};
```

Guard optional route params before fetching:

```typescript
// features/project/resolvers/get-by-id.resolver.ts
export const getByIdResolver: ResolveFn<ProjectDetailsDto | undefined> = (route, state) => {
  const id = route.paramMap.get('id') || null;
  if (id === null) return undefined;
  return inject(ProjectService).getById<ProjectDetailsDto>(id);
};
```

### Route Attachment

```typescript
{
  path: '',
  loadComponent: () => import('./list/list.component'),
  resolve: { projects: getAllProjectsResolver }
}
```

### Data Access in Component Constructor

```typescript
constructor() {
  private readonly activatedRoute = inject(ActivatedRoute);
  const data = this.activatedRoute.snapshot.data['projects'] as ProjectListItemDto[] || [];
  this.facade.setAllProjects(data);
}
```

Use `snapshot.data` (synchronous). Do not subscribe to `activatedRoute.data`.

### loadComponent Style

Prefer dynamic import for lazy loading:

```typescript
loadComponent: () => import('./list/list.component')   // preferred — lazy
loadComponent: () => CreateEditComponent               // eager at bundle level — use sparingly
```

## 6. Anti-Patterns

- Do not use named exports for routable feature pages or child route files.
- Do not create a global routes file under `src/app/features`; feature routing belongs either in `app.routes.ts` for top-level entries or in each feature's own `routes.ts` for child routes.
- Do not use `component:` for new standalone page routes.
- Do not create route paths that differ from feature folder names without a clear product reason.
- Do not subscribe inside a resolver — return the observable directly.
- Do not access the facade from a resolver — resolvers call the feature service only.
- Do not access resolver data via `activatedRoute.data` Observable subscription; use `snapshot.data` synchronously in the constructor.

## 7. Final Checklist

- Page component or nested routes file has a default export.
- Route imports the target from the feature folder.
- Route uses `loadComponent: () => PageClass` or `loadChildren: () => ChildRoutes`.
- Feature path is kebab-case.
- Navigation links, menus, or layout entries are updated when needed.
- Resolvers are defined as named `ResolveFn<T>` exports in `resolvers/` and attached via `resolve: {}`.
