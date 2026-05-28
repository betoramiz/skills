-----
name: facade
description: This skill provides guidelines and patterns for building service facades in modern Angular (v19+).
---

# Angular Modern Service Facades

This skill provides guidelines and patterns for creating Service Facades in modern Angular (v19+ and v21+) to manage component state, orchestrate REST API calls, and clean up asynchronous streams.

---

## 1. Injectable and Lifecycle binding

Service Facades act as local state controllers for features. They should be decorated with `@Injectable()` without `providedIn: 'root'`, allowing components to provide them locally so their lifecycle is tied directly to the component.

```typescript
import { Injectable, inject, DestroyRef } from '@angular/core';

@Injectable()
export class FeatureFacade {
  private readonly destroyRef = inject(DestroyRef);
}
```

---

## 2. Managing State with Signals

Facades expose state using read-only or read-write Angular Signals. This guarantees that UI components automatically react to state changes.

```typescript
import { signal } from '@angular/core';
import { ActionStatus } from '@shared/models/types';

export class FeatureFacade {
  // Action status tracker: 'idle' | 'loading' | 'success' | 'error'
  actionStatus = signal<ActionStatus>('idle');
  errorMessage = signal<string>('');
  
  // Feature data signal
  items = signal<Item[]>([]);
}
```

---

## 3. Orchestrating Requests & RxJS Pipeline

Facades wrap REST service calls. We distinguish between:
1. **Returning Streams**: Methods that return an `Observable` to let the component subscribe or resolve via async pipes.
2. **Local Internal Subscriptions**: Fire-and-forget operations where the Facade handles the subscription internally.

### Pattern: Internal Subscription with takeUntilDestroyed
Always use `takeUntilDestroyed(this.destroyRef)` before subscribing internally to prevent memory leaks when the component is destroyed.

```typescript
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { switchMap, tap, catchError } from 'rxjs/operators';

export class FeatureFacade {
  createItem(payload: NewItem): void {
    this.actionStatus.set('loading');

    this.featureService
      .create(payload)
      .pipe(
        // Chain with listing refresh if needed
        switchMap(() => this.featureService.getListOf()),
        tap(data => {
          this.items.set(data);
          this.actionStatus.set('success');
        }),
        catchError(error => this.handleError(error)),
        takeUntilDestroyed(this.destroyRef) // Crucial for cleanup
      )
      .subscribe();
  }
}
```

For detailed architectural facade patterns and RxJS orchestration strategies, see [references/facade-patterns.md](references/facade-patterns.md).
