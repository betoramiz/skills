-----
name: architecture
description: This skill provides a complete overview of the software architecture and directory conventions used in the project.
---

# Project Software Architecture & Structural Conventions

This skill provides a comprehensive structural guide to the layers, directory structures, naming conventions, and software patterns implemented in the project. Always adhere to these rules when adding new features, pages, or shared components.

---

## 1. Directory Structure Blueprint

The application is structured into domain-specific modules, layout shells, and a shared core module:

```
src/app/
├── app.config.ts                   # Global config (Routing, Providers)
├── app.routes.ts                   # Main route definition (Lazy loading)
├── app.ts                          # Root component
├── shared/                         # Common infrastructure (aliased as @shared)
│   ├── components/                 # Reusable UI widgets
│   │   ├── dialgos/                # Typos note: Shared dialogs (confirmation, etc.)
│   │   └── layout/                 # Structural containers (page-content, etc.)
│   ├── services/                   # App-wide services (BaseService, ModalService)
│   ├── models/                     # Shared pagination, types, DTO interfaces
│   └── directives/ | pipes/ | utils/
├── layouts/                        # Page-level master shells
│   ├── auth-layout-component/      # Unauthenticated wrapper
│   └── main-layout-component/      # Authenticated sidenav shell
└── dashboard-features/             # Feature-based domain modules
    ├── forms/                      # Forms module example
    │   ├── components/             # Inner domain components
    │   ├── models/                 # Feature-specific models & DTO mappers
    │   ├── form-service.ts         # REST API Service
    │   ├── form-facade.ts          # Feature State Controller
    │   └── forms.ts | html | css   # Page Component (Default export)
```

---

## 2. Naming Conventions

### Component Files
- **Layout components**: `<name>-component.ts` (e.g. `main-layout-component.ts`)
- **Shared Dialogs**: `<name>-component.ts` (e.g. `confirmation-component.ts`)
- **Page & Feature components**: `<name>.ts`, `<name>.html`, `<name>.css` (e.g. `forms.ts`, omitting the `.component` suffix for clean directory naming).

### Service Files
- **REST Services**: `<name>-service.ts` (e.g. `form-service.ts`)
- **Facades**: `<name>-facade.ts` (e.g. `form-facade.ts`)

---

## 3. Data and Logic Flow (The 3-Layer Pattern)

To maintain a clean separation of concerns, the application enforces a strict three-tier architecture for feature modules:

```
┌─────────────────────────────────┐
│        1. View Layer            │
│   (Page & Inner Components)     │
└────────────────┬────────────────┘
                 │ (Binds template to Signals, calls Actions)
                 ▼
┌─────────────────────────────────┐
│       2. Facade Layer           │
│       (Feature Facade)          │
└────────────────┬────────────────┘
                 │ (Manages UI state via Signals, orchestrates RxJS)
                 ▼
┌─────────────────────────────────┐
│       3. Data REST Layer        │
│   (Feature Service & HttpClient)│
└─────────────────────────────────┘
                 │ (Executes HTTP calls, extends BaseService)
                 ▼
          [ Backend API ]
```

### Layer Principles:
1. **View Layer**: Components are "dumb". They do not execute HTTP calls directly, nor do they manage complex RxJS pipelines. They inject their feature Facade, bind template elements to the Facade's Signals, and delegate user actions to it.
2. **Facade Layer**: Local feature controller (provided at component level). It exposes read-only or read-write signals representing UI states (`actionStatus`, `errorMessage`, data arrays). It orchestrates service calls and manages subscription lifecycles using `takeUntilDestroyed()`.
3. **Data REST Layer**: Stateless REST API wrapper extending `BaseService` and hitting the API endpoints.

---

## 4. Path Aliasing

Always import shared core modules using the `@shared` path mapping rather than relative paths:

```typescript
// Good Pattern
import { ConfirmationComponent } from '@shared/components/dialgos/confirmation-component';

// Avoid
import { ConfirmationComponent } from '../../shared/components/dialgos/confirmation-component';
```

## 5. UI and Theming Architecture

1. **Angular Material 3 Theme SCSS**: Global theming is maintained inside `src/themes/custom-theme.scss` using `@use '@angular/material' as mat`. Component structural changes must be done using specific Material mixin overrides (`mat.sidenav-overrides`, `mat.list-overrides`, etc.) instead of hardcoded CSS class hacks.
2. **Tailwind CSS v4 Utility Classes**: Used inside templates for structural styling, alignment, grid layouts, and responsiveness coexisting with Material components.
