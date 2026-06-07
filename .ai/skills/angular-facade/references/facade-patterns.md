# Facade Architectural & State Patterns

This document details the architectural design patterns, state management practices, and conventions utilized for Service Facades in modern Angular.

---

## 1. Architectural Role of a Facade

A **Facade** serves as the intermediate layer between UI components and backend REST services.

```
┌─────────────────┐       ┌────────────────┐       ┌─────────────────┐
│  UI Component   │ ◄───► │ Service Facade │ ◄───► │ Backend Service │
└─────────────────┘       └────────────────┘       └─────────────────┘
  (Displays UI &            (Manages state,          (Direct HTTP REST
   triggers events)          RxJS pipelines)          calls to API)
```

### Key Differences:
- **Feature Service**: Extends `BaseService`, is declared global with `providedIn: 'root'`, handles direct HTTP requests, and returns raw Observables.
- **Service Facade**: Extends `BaseCrudFacade`, is provided at the Component level, orchestrates business logic pipelines, inherits common reactive state signals (`actionStatus`, `errorMessage`), manages feature-specific data signals, and runs operations safely through `runQuery`.

---

## 2. Complete Facade Implementation Reference

Below is the established blueprint pattern for a modern state-managing feature Facade:

```typescript
import { inject, Injectable, signal } from '@angular/core';
import { BaseCrudFacade } from '@shared-services/base-crud-facade';
import { FormService } from './form-service';
import { switchMap } from 'rxjs';

@Injectable()
export class FormFacade extends BaseCrudFacade {
  // 1. Dependency injection and service mapping
  protected readonly service = inject(FormService);

  // 2. Feature-specific data state
  items = signal<string[]>([]);

  // 3. Simple Query Execution using runQuery
  getAllItems(): void {
    this.runQuery(
      this.service.getListOf<string>('list'),
      (data) => this.items.set(data)
    );
  }

  // 4. Setter for resolver-provided data (no runQuery needed)
  setAllItems(items: string[]): void {
    this.items.set(items);
  }

  // 5. Command pipeline orchestration with automatic state handling
  createItem(name: string, email: string): void {
    const pipeline$ = this.service
      .create<{ name: string; email: string }, any>({ name, email })
      .pipe(
        // SwitchMap to automatically fetch refreshed data
        switchMap(() => this.service.getListOf<string>('list'))
      );

    this.runQuery(
      pipeline$,
      (data) => this.items.set(data)
    );
  }
}
```

### Key Advantages of `BaseCrudFacade`:
1. **Inherited Signals**: `actionStatus` and `errorMessage` are inherited automatically and exposed as read-only or writable (depending on component requirements).
2. **Subscription Management**: `runQuery` internally handles subscription and uses `takeUntilDestroyed` with the facade's `DestroyRef` to prevent memory leaks.
3. **Consistent State Transitions**: The `setLoading()`, `setSuccess()`, and `handleError()` steps are executed automatically, keeping the UI state synchronized with the REST call lifecycle.

Inherited `create`, `update`, and `delete` return observables instead of subscribing internally. If a component or facade uses those methods directly, the caller must subscribe with an Angular-safe cleanup strategy such as `takeUntilDestroyed`.

---

## 3. Component Integration Pattern

When using the Facade inside a Component:
1. Include the Facade in the component's `providers` array.
2. Inject it using `inject()`.
3. Expose facade signals to templates with `.asReadonly()` to prevent template-side mutation.
4. Use `effect()` to handle both error **and** success status branches.

```typescript
import { Component, effect, inject } from '@angular/core';
import { FormFacade } from './form-facade';
import { ModalService } from '@shared-services/modal-service';
import { Router } from '@angular/router';

@Component({
  selector: 'app-forms-page',
  providers: [FormFacade],
  templateUrl: './forms-page.html',
  styleUrl: './forms-page.css',
})
export default class FormsPage {
  protected readonly facade = inject(FormFacade);
  private readonly modalService = inject(ModalService);
  private readonly router = inject(Router);

  // Expose signals as read-only — templates cannot mutate them
  protected readonly items  = this.facade.items.asReadonly();
  protected readonly status = this.facade.actionStatus.asReadonly();

  constructor() {
    // Handle both error and success branches
    effect(() => {
      if (this.status() === 'error') {
        this.modalService.showErrorModal(this.facade.errorMessage());
        this.facade.clearStatus();
      } else if (this.status() === 'success') {
        this.router.navigate(['/target-path']);
        this.facade.clearStatus();
      }
    });
  }
}
```

---

## 4. Resolver Data Hydration

When a route resolver preloads data, the page component receives it synchronously and passes it to the facade via a setter method. No `runQuery` call is needed for the initial load — the resolver already made the HTTP request.

### Facade setter
```typescript
setAllProjects(projects: ProjectListItemDto[]): void {
  this.projects.set(projects);
}
```

### Page component constructor
```typescript
constructor() {
  const data = inject(ActivatedRoute).snapshot.data['projects'] as ProjectListItemDto[] || [];
  this.facade.setAllProjects(data);
}
```

For user-triggered refreshes after a create or update, use `runQuery()` separately (e.g. `createAndRefresh()` pattern with `switchMap`).

---

## 5. Anti-Patterns

- Do not add `providedIn: 'root'` to feature facades.
- Do not subscribe in components when the facade can expose a `runQuery` method instead.
- Do not duplicate loading/error signals in feature facades unless the feature has multiple independent async regions.
- Do not put URL construction or raw `HttpClient` calls in facades.
- Do not leave subscriptions from returned CRUD observables without cleanup.
- Do not use `@shared/services/...` — use `@shared-services/...` (the granular alias).

## 6. Final Checklist

- Facade is provided by the owning component.
- Facade extends `BaseCrudFacade` and assigns `protected readonly service`.
- Async pipelines use `runQuery` when the facade owns the subscription.
- Feature data lives in signals.
- Components bind to facade signals via `.asReadonly()` and delegate actions to facade methods.
- If the page uses a resolver, the facade has a setter method to receive resolver data.
