---
name: angular-architecture
description: Apply this project's Angular application architecture. Use when adding or refactoring dashboard features, routable pages, shared layout components, feature services, facades, models, DTO mappers, route entries, folder structure, path aliases, Tailwind v4 layout, or Angular Material 3 theme conventions.
---

# Angular Architecture

Follow the project's feature-first structure and three-layer flow. Prefer these conventions over generic Angular defaults.

## Feature Structure

```text
src/app/features/<feature-name>/
├── components/
│   └── <inner-component>/
│       ├── <inner-component>.ts
│       ├── <inner-component>.html
│       └── <inner-component>.css
├── models/
│   ├── <feature-name>-models.ts
│   └── mappers.ts
├── <feature-name>-service.ts
├── <feature-name>-facade.ts
├── <feature-name>.ts
├── <feature-name>.html
└── <feature-name>.css
```

## Layer Rules

1. View layer: components bind to signals and delegate actions. They should not perform HTTP calls or complex RxJS orchestration.
2. Facade layer: feature-local controller extending `BaseCrudFacade` that inherits unified UI state signals (`actionStatus`, `errorMessage`) and uses query runners (`runQuery`) to orchestrate async pipelines and handle auto-unsubscription on destruction.
3. Data REST layer: stateless service extending `BaseService`, returning typed observables.


## Naming and Imports

- Routable pages use default exports.
- Nested components use named exports.
- Feature pages omit the `.component` suffix.
- Shared dialogs and layout components keep the existing `<name>-component.ts` convention.
- Shared imports use `@shared/...`.
- Imports inside the same feature use relative paths.
- The shared dialogs folder is currently spelled `dialgos`; preserve that path unless explicitly fixing the project-wide typo.

## UI Stack

- Use Tailwind CSS v4 utilities for layout, spacing, grid, and responsive structure.
- Use Angular Material 3 components and tokens for interactive controls.
- Put global Material overrides in `src/themes/custom-theme.scss`; avoid component-level CSS hacks against internal Material classes.

## Related Skills

- Use `angular-routing` when adding or changing route entries.
- Use `angular-testing` when adding tests for a feature, facade, service, route, or form.
- Use `angular-material-m3-theming` when changing Material theme tokens or override mixins.

## Anti-Patterns

- Do not put feature-specific components in `shared`.
- Do not call REST services directly from routable pages when a facade should own orchestration.
- Do not mix shared alias imports and deep relative imports for the same shared module.
- Do not rename the existing `dialgos` folder unless performing a project-wide migration.

## Final Checklist

- Feature follows the folder structure above.
- Page component is routable and default-exported.
- Feature service extends `BaseService`.
- Feature facade extends `BaseCrudFacade` and is provided by the page component.
- Shared imports use `@shared/...`.

Read [references/architecture-patterns.md](references/architecture-patterns.md) before scaffolding a new feature or changing project structure.
