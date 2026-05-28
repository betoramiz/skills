-----
name: service
description: This skill provides guidelines and patterns for building services in modern Angular (v19+).
---

# Angular Modern Service Design & REST APIs

This skill provides guidelines and patterns for creating services in modern Angular (v19+ and v21+) using dependency injection (`inject()`) and extending the shared `BaseService` for HTTP/REST communication.

---

## 1. Injectable Decorator

All services must be decorated with `@Injectable` and configured with `providedIn: 'root'` to ensure they are singletons and tree-shakable.

```typescript
import { Injectable } from '@angular/core';

@Injectable({
  providedIn: 'root',
})
export class CustomService {
}
```

---

## 2. Shared REST Services via BaseService

Any service that performs HTTP REST operations must inherit from `BaseService` (located in `@shared/services/base-service`) to gain out-of-the-box CRUD methods:
- `getById<T>(id)`
- `getListOf<T>()`
- `getPaginatedResults<T>(pagination)`
- `create<T, U>(payload)`
- `update<T, U>(payload)`
- `delete<T>(id)`

### Extending BaseService Pattern
Provide the backend endpoint name to the base class using the `super()` call in the constructor.

```typescript
import { Injectable } from '@angular/core';
import { BaseService } from '@shared/services/base-service';

@Injectable({
  providedIn: 'root',
})
export class FormService extends BaseService {
  constructor() {
    // Passes the endpoint name (e.g. 'users' maps to environment.apiUrl/users)
    super('users');
  }
}
```

---

## 3. Dependency Injection in Services

Like components, services should inject dependencies using the functional `inject()` API at the class-property level, rather than constructor injection.

```typescript
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

@Injectable({
  providedIn: 'root',
})
export class UserPreferencesService {
  // Good Pattern
  private readonly httpClient = inject(HttpClient);
  
  // Custom API method
  getSettings(): Observable<Settings> {
    return this.httpClient.get<Settings>('/api/settings');
  }
}
```

For detailed architectural service patterns and configuration tips, see [references/service-patterns.md](references/service-patterns.md).
