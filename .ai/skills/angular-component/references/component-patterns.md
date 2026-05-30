# Component & Architectural Patterns

This document details the established architectural design patterns, conventions, and practices utilized across the Angular codebase. Follow these guidelines when creating or refactoring components and services.

## 1. File Naming & Folder Structure

To keep the workspace clean and modular, the codebase follows specific naming and structuring conventions.

### Component File Naming
Unlike the standard Angular CLI convention of suffixing files with `.component` (e.g., `modal.component.ts`), this repository omits the `.component` suffix for feature and layout components.
- **Page & Feature Components**: `<name>.ts`, `<name>.html`, `<name>.css` (e.g., `forms.ts`, `forms.html`, `forms.css`)
- **Layout Components**: `<name>-component.ts` (e.g., `main-layout-component.ts`)
- **Shared Dialogs**: `<name>-component.ts` (e.g., `confirmation-component.ts` under `@shared/components/dialgos/`)

### Feature Directories
Feature components are organized under `src/app/features/` inside their own directories. If a component uses child/nested components, they are placed in a sub-folder named `components/`.
```
src/app/features/forms/
├── components/                  # Nested, reusable components
│   ├── simple-form/
│   │   ├── simple-form.ts
│   │   ├── simple-form.html
│   │   └── simple-form.css
│   └── verbose-form/
│       ├── verbose-form.ts
│       ├── verbose-form.html
│       └── verbose-form.css
├── models/                      # Feature-specific models & mappers
│   ├── form-models.ts
│   └── mappers.ts
├── forms.ts                     # Main feature page component
├── forms.html
└── forms.css
```

---

## 2. Routable vs. Nested Components

We distinguish between components used as **route targets (pages/layouts)** and **reusable/nested elements**.

### A. Routable Components (Pages & Layouts)
- **Exports**: Must use a **default export** (`export default class ClassName`).
- **Routing**: Eagerly imported but declared in routes via the dynamic load function.
  ```typescript
  // src/app/features/routes.ts
  import Forms from './forms/forms';
  
  export const routes: Routes = [
    {
      path: 'forms',
      loadComponent: () => Forms
    }
  ]
  ```

### B. Reusable & Nested Components
- **Exports**: Must use **named exports** (`export class ClassName`).
- **Imports**: Statically imported by parent components.
  ```typescript
  import { SimpleForm } from './components/simple-form/simple-form';
  ```

---

## 3. Component Code Patterns

All components must adhere to the modern standalone API and reactive standards.

### Standalone Components
All components are standalone. Specify all component and material dependencies explicitly in the `imports` array:
```typescript
@Component({
  selector: 'app-my-feature',
  imports: [
    PageContent,
    PageHeader,
    PageBody,
    MatButton,
    MyNestedComponent
  ],
  templateUrl: './my-feature.html',
  styleUrl: './my-feature.css'
})
export default class MyFeature {}
```

### Dependency Injection (DI)
Use Angular’s functional `inject()` API instead of constructor injection. Injected properties should be defined at the class property initialization level.
```typescript
// Good Pattern
private readonly modalService = inject(ModalService);
private readonly fb = inject(NonNullableFormBuilder);

// Avoid
constructor(private modalService: ModalService) {}
```

### Single-File Components (SFC)
For small components, utility views, and layouts (e.g., `<page-header>`, custom wrappers, simple dialogs), use inline templates and styles inside the `@Component` decorator with backtick syntax:
```typescript
@Component({
  selector: 'page-content',
  template: `
    <div class="page-container">
      <ng-content></ng-content>
    </div>
  `,
  styles: `
    .page-container {
      margin: 12px 32px;
    }
  `,
})
export class PageContent {}
```

### Reactivity with Signals
Manage local component state using Angular **Signals**. Keep property accessors clean and direct:
```typescript
import { signal, computed, effect } from '@angular/core';

// Define state signal
backendDto = signal<BackendDto | null>(null);

// Update signal
this.backendDto.set(newData);

// Derived computed state
hasData = computed(() => this.backendDto() !== null);
```

### Modern Component Inputs as Signals
Always use Signal Inputs instead of the old `@Input()` decorator.
```typescript
import { input, computed } from '@angular/core';

// Optional input with default value
label = input<string>('Save');

// Required input
userId = input.required<string>();

// Computed value based on input signal
userGreeting = computed(() => `Hello User ${this.userId()}`);
```

### Modern Component Outputs
Use the modern functional `output()` API instead of `@Output() / EventEmitter`:
```typescript
import { output } from '@angular/core';

// Good Pattern
onSubmit = output<FormValueType>();

// Emitting
this.onSubmit.emit(this.formGroup.getRawValue());
```


---

## 4. Forms & DTO Mapping

We enforce type safety for forms and clean separation of DTOs (Data Transfer Objects) and UI models.

### Reactive Forms with NonNullableFormBuilder
1. Inject `NonNullableFormBuilder` to enforce non-nullable value types.
2. Build the form in a private helper method and assign it to a `protected readonly` property:
```typescript
export class SimpleForm {
  private readonly fb = inject(NonNullableFormBuilder);
  
  // Protect formGroup from external modifications but allow template access
  protected readonly formGroup = this.buildForm();
  
  private buildForm() {
    return this.fb.group({
      name: ['', Validators.required],
      email: ['', [Validators.required, Validators.email]]
    });
  }
}
```

### Model Mappings & TypeScript Utility Types

To ensure full type safety and modular architecture, keep form controls/values, API request payloads, and UI representations clearly defined. Place these in `models/<feature>-model.ts` inside the feature directory, utilizing TypeScript utility types (`Omit` and `Required`) to prevent duplicate property definitions for creation and editing, alongside separate interfaces for lists and details.

#### 1. Form Value (`FormValue`)
Represents the exact, raw fields bound to the component reactive form.
```typescript
export interface TimesheetFormValue {
  task: string;
  hours: number;
  date: string;
}
```

#### 2. Base DTO / Entity Type (`AddEditDto` / `AddEditEntity`)
Represents the fully populated entity used for both adding and editing. Contains the primary key (`id`) and all other relevant fields.
```typescript
export interface AddEditTimeSheet {
  id: number;
  projectId: number;
  task: string;
  hours: number;
  date: Date;
}
```

#### 3. Specialized Add and Edit Request Payloads
Leverage TypeScript's `Omit` and `Required` utility types to dynamically derive creation and edit requests from the base DTO:
- **Creation Payload (`AddRequest`)**: Omit the `id` property using `Omit`, since IDs are generated by the server.
- **Editing Payload (`EditRequest`)**: Require the `id` and all other fields using `Required`.
```typescript
export type AddTimeSheetRequest = Omit<AddEditTimeSheet, 'id'>;
export type EditTimeSheetRequest = Required<AddEditTimeSheet>;
```

#### 4. UI Representation Interfaces (List and Detail)
Explicitly separate the models used for listing vs. detail views to keep views lightweight:
- **List Item (`ListItemDto` / `<Entity>ListItem`)**: For tables/grids, containing only columns to display.
- **Detail View (`DetailsDto` / `<Entity>Details`)**: For full page views or comprehensive detail cards.
```typescript
export interface EntryListItem {
  id: number;
  task: string;
  hours: number;
  date: Date;
}
```

#### 5. Mapper Implementations (`mappers.ts`)
Store mapper functions in a sibling `mappers.ts` file under the feature's `models/` directory. This keeps mapping logic separated from both components and services:
```typescript
import { AddTimeSheetRequest, TimesheetFormValue } from './timesheet-model';

export function fromFormToCreateDto(projectId: number, data: TimesheetFormValue): AddTimeSheetRequest {
  return {
    task: data.task,
    hours: data.hours,
    projectId: projectId,
    date: new Date(data.date),
  };
}
```

---

## 5. Shared Services & Infrastructure

### BaseService for HTTP REST Calls
Any service that performs HTTP operations should inherit from `BaseService` (located in `@shared/services/base-service.ts`) which encapsulates standard CRUD endpoints.
- **Injecting Base HttpClient**: Even though `BaseService` is an inheritance-based class, it resolves `HttpClient` via `inject()` property assignment:
  ```typescript
  protected httpClient: HttpClient = inject(HttpClient);
  ```
- **Gotchas / Internal Quirks**:
  - The base service stores its configured endpoint in `_baseUrl`, exposed cleanly through the protected `baseUrl` getter.
  - The endpoint is configured via the constructor: `super('my-endpoint')`.

### Centralized Modal Service
Interact with dialogs via `ModalService` (`@shared/services/modal-service.ts`).
- **Gotcha**: Note that the shared dialogs directory has a small folder naming typo: `@shared/components/dialgos` (spelled `dialgos` instead of `dialogs`). All shared confirmation, error, and yes/no components are stored there.

---

## 6. Styling, Path Mappings, and UI Design

- **Tailwind CSS v4**: Built-in styling uses Tailwind CSS v4. Standard utility classes (`flex`, `gap-5`, `text-xl`, etc.) are heavily leveraged in templates.
- **Path Aliasing**: Always import shared code using the `@shared` alias to prevent deep relative path nesting:
  ```typescript
  import { PageBody } from '@shared/components/layout';
  ```
  Defined in `tsconfig.json` as:
  ```json
  "paths": {
    "@shared/*": ["src/app/shared/*"]
  }
  ```
- **Aesthetic Guidelines**: Custom designs must look premium, modern, and aligned with standard layout utilities (`<page-content>`, `<page-header>`, `<page-body>`). Use curated colors (such as Angular Material tokens and HSL palettes) instead of default browser styles.

---

## 7. Anti-Patterns

- Do not use constructor parameter injection for new code.
- Do not use `@Input()`, `@Output()`, or `EventEmitter` for new components.
- Do not import shared code through deep relative paths; use `@shared/...`.
- Do not put HTTP calls or multi-step RxJS orchestration directly in components.
- Do not target generated Angular Material DOM classes from component CSS.

## 8. Final Checklist

- Component is standalone and all template dependencies are listed in `imports`.
- Feature pages use default exports; nested components use named exports.
- Templates use `@if`, `@for`, and `@switch`.
- Shared imports use `@shared/...`; same-feature imports are relative.
- Layout remains responsive on mobile and desktop widths.
