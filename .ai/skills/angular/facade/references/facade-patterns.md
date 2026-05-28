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
import { BaseCrudFacade } from '@shared/services/base-crud-facade';
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

  // 4. Command pipeline orchestration with automatic state handling
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
3. Bind the template or effects directly to the facade's state and data signals.

```typescript
import { Component, effect, inject, OnInit } from '@angular/core';
import { FormFacade } from './form-facade';
import { ModalService } from '@shared/services/modal-service';

@Component({
  selector: 'app-forms-page',
  providers: [FormFacade], // Tied to this component's lifecycle
  template: `
    @if (facade.actionStatus() === 'loading') {
      <mat-spinner></mat-spinner>
    }

    @if (facade.errorMessage()) {
      <div class="error">{{ facade.errorMessage() }}</div>
    }

    <button (click)="facade.createItem('Alice', 'alice@email.com')">Add User</button>

    <ul>
      @for (item of facade.items(); track item) {
        <li>{{ item }}</li>
      }
    </ul>
  `
})
export default class FormsPage implements OnInit {
  protected readonly facade = inject(FormFacade);
  private readonly modalService = inject(ModalService);

  constructor() {
    // Standard effect for global/modal notifications on errors
    effect(() => {
      if (this.facade.actionStatus() === 'error') {
        this.modalService.showErrorModal(this.facade.errorMessage());
        this.facade.clearStatus(); // Clean up state after displaying
      }
    });
  }

  ngOnInit() {
    this.facade.getAllItems();
  }
}
```

---

## 4. Anti-Patterns

- Do not add `providedIn: 'root'` to feature facades.
- Do not subscribe in components when the facade can expose a `runQuery` method instead.
- Do not duplicate loading/error signals in feature facades unless the feature has multiple independent async regions.
- Do not put URL construction or raw `HttpClient` calls in facades.
- Do not leave subscriptions from returned CRUD observables without cleanup.

## 5. Final Checklist

- Facade is provided by the owning component.
- Facade extends `BaseCrudFacade` and assigns `protected readonly service`.
- Async pipelines use `runQuery` when the facade owns the subscription.
- Feature data lives in signals.
- Components bind to facade signals and delegate actions to facade methods.
