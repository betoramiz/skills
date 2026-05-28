# Architecture & Modular Scaffolding Reference

This document details the architectural conventions, file naming standards, path aliases, and structural flow patterns enforced across the Angular codebase.

---

## 1. Feature Module Scaffolding Blueprint

When creating a new domain feature under `src/app/dashboard-features/`, follow this exact folder structure:

```
src/app/dashboard-features/<feature-name>/
├── components/                      # Private, domain-specific components
│   └── <inner-component>/
│       ├── <inner-component>.ts
│       ├── <inner-component>.html
│       └── <inner-component>.css
├── models/                          # Feature data definitions
│   ├── <feature-name>-models.ts     # TypeScript interfaces and types
│   └── mappers.ts                   # DTO mapping conversion functions
├── <feature-name>-service.ts        # REST service extending BaseService
├── <feature-name>-facade.ts         # Controller managing signals and subscriptions
├── <feature-name>.ts                # Main page entry component
├── <feature-name>.html
└── <feature-name>.css
```

---

## 2. Shared Core Module (`src/app/shared/`)

The shared module is a central directory for reusable utilities, UI layouts, types, and services. It is divided into logical subfolders:

- `components/layout/`: Common wrappers and page layouts, such as `page-content`, `page-header`, `page-body`.
- `components/dialgos/`: Centralized modal widgets (e.g. `confirmation-component.ts`, `error-component.ts`, `yes-no-component.ts`). **Notice the folder name spelling: "dialgos"**.
- `services/`: Infrastructure utilities like `BaseService` (REST core class) and `ModalService` (dialog trigger coordinator).
- `models/`: App-wide structures, such as `Pagination` and common type declarations.

---

## 3. Path Mapping / Aliases

The project utilizes TypeScript path mapping defined in `tsconfig.json`. Absolute aliases are preferred over relative paths:

```json
"paths": {
  "@shared/*": ["src/app/shared/*"]
}
```

### Import Rules:
- **Shared items**: Always use `@shared/...`.
- **Feature items**: Use relative imports (`./components/...`, `./models/...`) when importing files *within the same feature module*.

---

## 4. UI Layout & Styles Stack

The application blends **Tailwind CSS v4** and **Angular Material 3 (M3)** using a unified styles system:

1. **Global Stylesheet**: `src/styles.css` imports Tailwind directives using `@import "tailwindcss";`.
2. **Material Custom Theme**: `src/themes/custom-theme.scss` loads `@angular/material` as `mat`, sets global typography to Roboto, and handles style overrides using dedicated design tokens and maps.
3. **Template Grid & Flexbox**: Components must use Tailwind grid and flex utilities (`grid`, `flex`, `gap-4`, `p-6`) for fast, clean, responsive positioning.
