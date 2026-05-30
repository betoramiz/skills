---
name: angular-testing
description: Add or refactor tests for this Angular dashboard. Use when testing standalone components, signal inputs and outputs, facades, BaseService subclasses, typed reactive forms, route entries, Angular Material interactions, HttpClient calls, or regressions in src/app.
---

# Angular Testing

Add focused tests around the changed behavior. Prefer small tests that verify public behavior over snapshots or implementation details.

## Core Rules

- Use Angular TestBed for standalone components and include the component in `imports`.
- Test signal inputs by setting inputs through the fixture/component API available in the project version.
- For form components, assert invalid submit behavior, emitted values from `getRawValue()`, and validation error states.
- For facades, mock the feature service and assert signal state transitions.
- For REST services, use Angular HTTP testing utilities and assert method, URL, body, and error paths.
- Keep tests close to the source file unless the project establishes a different convention.
- Run `npm test` or a narrower available test command after adding tests.

Read [references/testing-patterns.md](references/testing-patterns.md) before adding tests for services, facades, or forms.
