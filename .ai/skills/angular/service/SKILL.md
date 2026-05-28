---
name: angular-service
description: Build or refactor Angular services for this project. Use when creating Injectable services, REST API wrappers, BaseService subclasses, custom HTTP endpoints, RxJS-returning service methods, or shared service infrastructure.
---

# Angular Services

Services are stateless data or infrastructure wrappers. Keep subscriptions out of services unless the service explicitly owns local caching state.

## Core Rules

- Decorate app-wide and feature data services with `@Injectable({ providedIn: 'root' })`.
- Use functional `inject()` for dependencies.
- Return `Observable<T>` from service methods; let facades or components subscribe.
- Any REST service should extend `BaseService` from `@shared/services/base-service`.
- Configure the endpoint by calling `super('<endpoint>')` in the constructor.
- Use `this.buildUrl(endpointName?, id?)` to construct consistent sub-endpoints and API paths instead of manually concatenating paths.
- Add `catchError(this.handleError)` for custom HTTP calls that bypass inherited CRUD helpers.

```typescript
import { Injectable } from '@angular/core';
import { Observable, catchError } from 'rxjs';
import { BaseService } from '@shared/services/base-service';
import { UserDto } from './models/user-models';

@Injectable({
  providedIn: 'root',
})
export class UserService extends BaseService {
  constructor() {
    super('users');
  }

  getActiveUsers(): Observable<UserDto[]> {
    // Hits: ${this.baseUrl}/active
    return this.getListOf<UserDto>('active');
  }

  updateStatus(id: string, active: boolean): Observable<void> {
    // Hits: ${this.baseUrl}/${id}/status
    const url = this.buildUrl(`${id}/status`);
    return this.httpClient
      .patch<void>(url, { active })
      .pipe(catchError(this.handleError));
  }
}
```

## Placement

- Shared infrastructure services live under `src/app/shared/services/`.
- Feature-specific REST services live under `src/app/dashboard-features/<feature>/`.

Read [references/service-patterns.md](references/service-patterns.md) when adding custom endpoints, extending `BaseService`, or deciding whether logic belongs in a service or facade.

