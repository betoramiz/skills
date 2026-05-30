# Testing Patterns

## 1. Standalone Component Test

```typescript
await TestBed.configureTestingModule({
  imports: [UserFormComponent],
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

## 5. Anti-Patterns

- Do not test private helpers directly.
- Do not assert implementation-only signal names when a user-facing DOM assertion is clearer.
- Do not leave pending HTTP requests in service tests.
- Do not use real network calls.
- Do not add broad brittle snapshots for Angular Material DOM.

## 6. Final Checklist

- Tests compile with standalone imports.
- Async observables are flushed or completed.
- HTTP mocks verify no outstanding requests.
- Form tests cover invalid and valid submit.
- Facade tests cover success and error paths.
