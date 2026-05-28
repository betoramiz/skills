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
Feature components are organized under `src/app/dashboard-features/` inside their own directories. If a component uses child/nested components, they are placed in a sub-folder named `components/`.
```
src/app/dashboard-features/forms/
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
  // src/app/dashboard-features/routes.ts
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

### Model Mappings
Keep form controls, form values, and DTO types separated. Create a `mappers.ts` inside the `models/` folder to handle conversions.
```typescript
// models/form-models.ts
export interface FormValueType {
  name: string;
  email: string;
}

export interface BackendDto {
  id?: number;
  name: string;
  email: string;
}

// models/mappers.ts
export const toBackendDto = (formValue: FormValueType): BackendDto => ({
  name: formValue.name,
  email: formValue.email
});
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
  - The base service uses an internal private variable named `baseUlr` (note the spelling typo `Ulr` instead of `Url`), which is exposed cleanly via the `protected get baseUrl()` getter.
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
