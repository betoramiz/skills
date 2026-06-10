---
name: angular-facade
description: Build or refactor Angular feature facades for this project. Use when adding component-scoped state controllers, orchestrating Angular service calls, managing signals for UI state, handling RxJS pipelines, or extending BaseCrudFacade.
---

# Angular Facades

Facades are feature-local state controllers between components and REST services. They own UI state, orchestration, and internal subscriptions.

## Core Rules

- Decorate facades with `@Injectable()` only; do not use `providedIn: 'root'`.
- Extend the reusable `BaseCrudFacade` from `@shared-services/base-crud-facade` to automatically inherit `actionStatus` and `errorMessage` signals.
- Provide the facade in the owning component's `providers` array.
- Inject the feature service and assign it to the required `protected readonly service` property.
- Use `this.runQuery(observable$, (data) => ...)` to execute query and command pipelines. This automatically sets `actionStatus` ('loading', 'success', 'error'), handles error propagation, and cleans up subscriptions using `takeUntilDestroyed`.
- If using inherited `create`, `update`, or `delete` methods directly, return the observable to the caller and ensure the caller owns subscription cleanup.
- Use signals for additional feature-specific UI state such as lists, selected items, filters, and pagination.
- Keep service classes stateless and let the facade coordinate multi-step flows.
- When a route resolver preloads data, receive it via `inject(ActivatedRoute).snapshot.data['key']` in the component constructor and pass it to a facade setter method. No `runQuery` needed for resolver-provided data.
- Use `effect()` for both error and success status branches: show modal on error, navigate on success, call `facade.clearStatus()` in both.
- Signals that hold API-sourced data must use DTO types (from `models/<feature>-dtos.ts`), not UI model types. UI model types (from `<feature>-models.ts`) are for form values and component-local state only.

```typescript
import { Injectable, inject, signal } from '@angular/core';
import { BaseCrudFacade } from '@shared-services/base-crud-facade';
import { UserService } from './user-service';
import { UserListDto } from './models/user-dtos';
import { UserFormValue } from './models/user-models';
import { switchMap } from 'rxjs';

@Injectable()
export class UserFacade extends BaseCrudFacade {
  // Required implementation of abstract service property
  protected readonly service = inject(UserService);

  // Additional feature-specific UI state — use DTO type for API-sourced lists
  users = signal<UserListDto[]>([]);

  loadUsers(): void {
    this.runQuery(
      this.service.getActiveUsers(),
      (users) => this.users.set(users)
    );
  }

  createUser(value: UserFormValue): void {
    const createAndRefresh$ = this.service
      .create<UserListDto, UserFormValue>(value)
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
