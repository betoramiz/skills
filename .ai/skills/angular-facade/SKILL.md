---
name: angular-facade
description: Build or refactor Angular feature facades for this project. Use when adding component-scoped state controllers, orchestrating Angular service calls, managing signals for UI state, handling RxJS pipelines, or extending BaseCrudFacade.
---

# Angular Facades

Facades are feature-local state controllers between components and REST services. They own UI state, orchestration, and internal subscriptions.

## Core Rules

- Decorate facades with `@Injectable()` only; do not use `providedIn: 'root'`.
- Extend the reusable `BaseCrudFacade` from `@shared/services/base-crud-facade` to automatically inherit `actionStatus` and `errorMessage` signals.
- Provide the facade in the owning component's `providers` array.
- Inject the feature service and assign it to the required `protected readonly service` property.
- Use `this.runQuery(observable$, (data) => ...)` to execute query and command pipelines. This automatically sets `actionStatus` ('loading', 'success', 'error'), handles error propagation, and cleans up subscriptions using `takeUntilDestroyed`.
- If using inherited `create`, `update`, or `delete` methods directly, return the observable to the caller and ensure the caller owns subscription cleanup.
- Use signals for additional feature-specific UI state such as lists, selected items, filters, and pagination.
- Keep service classes stateless and let the facade coordinate multi-step flows.

```typescript
import { Injectable, inject, signal } from '@angular/core';
import { BaseCrudFacade } from '@shared/services/base-crud-facade';
import { UserService } from './user-service';
import { UserDto, UserFormValue } from './models/user-models';
import { switchMap } from 'rxjs';

@Injectable()
export class UserFacade extends BaseCrudFacade {
  // Required implementation of abstract service property
  protected readonly service = inject(UserService);

  // Additional feature-specific UI state
  users = signal<UserDto[]>([]);

  loadUsers(): void {
    this.runQuery(
      this.service.getActiveUsers(),
      (users) => this.users.set(users)
    );
  }

  createUser(value: UserFormValue): void {
    const createAndRefresh$ = this.service
      .create<UserDto, UserFormValue>(value)
      .pipe(
        switchMap(() => this.service.getActiveUsers())
      );

    this.runQuery(
      createAndRefresh$,
      (users) => this.users.set(users)
    );
  }
}
```

Read [references/facade-patterns.md](references/facade-patterns.md) for full facade/component integration and RxJS orchestration examples.
