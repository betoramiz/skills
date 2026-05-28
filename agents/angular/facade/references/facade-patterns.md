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
- **Service Facade**: Is provided at the Component level, orchestrates business logic pipelines, manages reactive state signals (`actionStatus`, `errorMessage`, data signals), and subscribes internally using clean lifecycle destruction.

---

## 2. Complete Facade Implementation Reference

Below is the established blueprint pattern for a modern state-managing feature Facade:

```typescript
import { DestroyRef, inject, Injectable, signal } from '@angular/core';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { catchError, Observable, switchMap, tap, throwError } from 'rxjs';
import { ActionStatus } from '@shared/models/types';
import { FormService } from './form-service';

@Injectable()
export class FormFacade {
  // 1. Dependencies resolved via property inject()
  private readonly formService = inject(FormService);
  private readonly destroyRef = inject(DestroyRef);

  // 2. Reactive UI State Signals
  actionStatus = signal<ActionStatus>('idle');
  errorMessage = signal<string>('');
  items = signal<string[]>([]);

  // 3. Routable Stream Method (letting component subscribe)
  getAllItems(): Observable<string[]> {
    this.actionStatus.set('loading');

    return this.formService
      .getListOf<string>('list')
      .pipe(
        tap((data) => {
          this.items.set(data);
          this.actionStatus.set('success');
        }),
        catchError(error => this.handleError(error))
      );
  }

  // 4. Fire-and-Forget Internal Subscription Method
  createItem(name: string, email: string): void {
    this.actionStatus.set('loading');

    this.formService
      .create({ name, email })
      .pipe(
        // SwitchMap to automatically fetch refreshed data
        switchMap(() => this.formService.getListOf<string>('list')),
        tap((data) => {
          this.items.set(data);
          this.actionStatus.set('success');
        }),
        catchError(error => this.handleError(error)),
        // Crucial: Automatically unsubscribes on component destruction
        takeUntilDestroyed(this.destroyRef)
      )
      .subscribe();
  }

  // 5. Utility State Reset Methods
  clearStatus(): void {
    this.actionStatus.set('idle');
    this.errorMessage.set('');
  }

  // 6. Centralized Internal Error Handler
  private handleError(error: any): Observable<never> {
    this.actionStatus.set('error');
    this.errorMessage.set(error.message || 'An error occurred');
    return throwError(() => error);
  }
}
```

---

## 3. Component Integration Pattern

When using the Facade inside a Component:
1. Include the Facade in the component's `providers` array.
2. Inject it using `inject()`.
3. Bind template directly to facade state signals.

```typescript
import { Component, inject, OnInit } from '@angular/core';
import { FormFacade } from './form-facade';

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
  `
})
export default class FormsPage implements OnInit {
  protected readonly facade = inject(FormFacade);

  ngOnInit() {
    this.facade.getAllItems().subscribe();
  }
}
```
