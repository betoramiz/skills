# Testing Patterns

## 1. Standalone Component Test

```typescript
await TestBed.configureTestingModule({
  imports: [UserFormComponent],
  providers: [provideZonelessChangeDetection()],  // match production config
}).compileComponents();

const fixture = TestBed.createComponent(UserFormComponent);
fixture.detectChanges();
```

## 2. Form Component Coverage

Cover these behaviors:

- Invalid submit calls `markAllAsTouched()` behavior and emits nothing.
- Valid submit emits the typed `getRawValue()` payload.
- Cancel emits the cancel output and never submits the form.
- Required and domain validation messages appear after touched/dirty state.

## 3. Facade Coverage

Mock the feature service with observable-returning methods. Assert:

- `actionStatus` becomes `loading` while a query starts.
- Successful query writes feature data signals and ends in `success`.
- Failed query writes `errorMessage` and ends in `error`.
- Chained operations refresh the expected list.

## 4. Service Coverage

Use HTTP testing utilities to verify:

- Base endpoint from `super('<endpoint>')`.
- Custom sub-endpoints through `buildUrl()`.
- Request method and payload.
- Error path is surfaced to the subscriber.

## 5. Resolver Coverage

Resolvers are plain `ResolveFn<T>` functions. Test them directly using `TestBed.runInInjectionContext()`:

```typescript
import { TestBed } from '@angular/core/testing';
import { ActivatedRouteSnapshot, RouterStateSnapshot, convertToParamMap } from '@angular/router';
import { of } from 'rxjs';
import { getAllProjectsResolver } from './get-all-projects.resolver';

it('returns projects from the service', () => {
  const fakeProjects = [{ id: 1, name: 'Test' }];
  const mockService = { getListOf: () => of(fakeProjects) };

  TestBed.configureTestingModule({
    providers: [
      provideZonelessChangeDetection(),
      { provide: ProjectService, useValue: mockService },
    ],
  });

  const fakeRoute = { paramMap: convertToParamMap({}) } as ActivatedRouteSnapshot;
  const result = TestBed.runInInjectionContext(() =>
    getAllProjectsResolver(fakeRoute, {} as RouterStateSnapshot)
  );

  expect(result).toBeDefined();
});
```

For optional-param resolvers, also assert that `undefined` is returned when the param is absent.

## 6. Anti-Patterns

- Do not test private helpers directly.
- Do not assert implementation-only signal names when a user-facing DOM assertion is clearer.
- Do not leave pending HTTP requests in service tests.
- Do not use real network calls.
- Do not add broad brittle snapshots for Angular Material DOM.

## 7. Final Checklist

- Tests compile with standalone imports.
- Async observables are flushed or completed.
- HTTP mocks verify no outstanding requests.
- Form tests cover invalid and valid submit.
- Facade tests cover success and error paths.
