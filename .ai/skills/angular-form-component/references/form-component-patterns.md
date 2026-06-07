# Form Component Architectural & UI Patterns

This document details the architectural design patterns, type definitions, and template layouts for form-type components in Angular.

---

## 1. Simple vs. Verbose Typing Patterns

Our codebase supports two levels of typing based on form complexity:

### A. Simple Form Typing
Best for basic forms where control structures are straightforward. The return type of `buildForm()` is inferred automatically by the compiler.
```typescript
export class SimpleForm {
  private readonly fb = inject(NonNullableFormBuilder);
  protected readonly formGroup = this.buildForm();

  private buildForm() {
    return this.fb.group({
      name: ['', Validators.required],
      email: ['', [Validators.required, Validators.email]],
    });
  }
}
```

### B. Verbose Form Typing (Recommended for Production)
Best for large forms or forms passed as references to other features. Requires explicit type mappings in a dedicated `models/form-models.ts` file.
```typescript
import { FormValueGroup } from '../../models/form-models';

export class VerboseForm {
  private readonly fb = inject(NonNullableFormBuilder);
  protected readonly formGroup = this.buildForm();

  private buildForm(): FormValueGroup {
    return this.fb.group({
      name: ['', Validators.required],
      email: ['', [Validators.required, Validators.email]],
    });
  }
}
```

---

## 2. HTML Template & Angular Material Layouts

Use semantic HTML tags and leverage standard Angular Material component forms inside form templates.
The `#ngForm="ngForm"` reference variable is required for full form reset (see Section 5).

```html
<form [formGroup]="formGroup" #ngForm="ngForm" (ngSubmit)="onSave()" class="flex flex-col gap-4 p-6">
  <!-- Name Input -->
  <mat-form-field class="w-full">
    <mat-label>Nombre</mat-label>
    <input matInput formControlName="name" placeholder="Ej. Juan Pérez" autocomplete="name">
    @if (formGroup.controls.name.invalid && formGroup.controls.name.touched) {
      <mat-error>El nombre es obligatorio</mat-error>
    }
  </mat-form-field>

  <!-- Email Input -->
  <mat-form-field class="w-full">
    <mat-label>Correo Electrónico</mat-label>
    <input matInput formControlName="email" type="email" placeholder="ejemplo@correo.com" autocomplete="email">
    @if (formGroup.controls.email.invalid && formGroup.controls.email.touched) {
      <mat-error>Introduce un correo válido</mat-error>
    }
  </mat-form-field>

  <!-- Actions -->
  <div class="flex justify-end gap-2.5 mt-4">
    <button matButton="text" type="button" (click)="cancel()">Cancelar</button>
    <button matButton="filled" type="submit" [disabled]="formGroup.invalid">Guardar</button>
  </div>
</form>
```

Do not set `appearance=""` on `mat-form-field` — the global config (`MAT_FORM_FIELD_DEFAULT_OPTIONS`) sets `appearance: 'outline'` project-wide.

---

## 3. Form Validation and Styling Guidelines

- **Form States**: Ensure validation triggers are responsive. Check validation with `formGroup.controls.controlName.touched` or `formGroup.controls.controlName.dirty` before rendering errors to prevent early red markings on pristine fields.
- **Specific Errors**: Prefer `hasError('required')`, `hasError('email')`, and custom validation keys over a generic `invalid` message when multiple failures are possible.
- **Browser Semantics**: Use native attributes such as `type="email"`, `autocomplete="email"`, `min`, `max`, and `maxlength` when they match the domain.
- **Tailwind Grid Alignment**: Arrange form fields in grid structures (`grid grid-cols-1 md:grid-cols-2 gap-4`) for spacious layouts.
- **Button Styling**: Always set `type="button"` on the Cancel button to prevent it from accidentally submitting the HTML form. Set `type="submit"` on the primary save action. Use `matButton="filled"` for primary actions and `matButton="text"` for secondary ones.

---

## 4. Edit Form Initialization

For an edit form, accept a typed input value and hydrate the form once the value exists. Use `setValue` when the incoming model is complete and `patchValue` only for intentionally partial objects.

```typescript
user = input<UserFormValue | null>(null);

constructor() {
  effect(() => {
    const user = this.user();
    if (user) {
      this.formGroup.setValue(user);
    }
  });
}
```

---

## 5. Form Reset Patterns

### Simple reset (values only)
```typescript
this.formGroup.reset()
```
Resets control values to initial state but does **not** reset Angular Material's touched/dirty state — fields will still show error styling after this call alone.

### Full reset (values + Material field state)
Required after a successful submit when the form should appear completely clean for the next entry.

**Template** — add the `#ngForm="ngForm"` template reference:
```html
<form [formGroup]="formGroup" #ngForm="ngForm" (ngSubmit)="onSave()">
```

**Class** — declare the viewChild signal and use the exact reset order:
```typescript
protected readonly ngForm = viewChild.required<FormGroupDirective>('ngForm');

onSave(): void {
  if (this.formGroup.invalid) {
    this.formGroup.markAllAsTouched();
    return;
  }
  const data = this.formGroup.getRawValue();   // 1. capture value BEFORE reset
  this.formGroup.reset();                       // 2. reset form control values
  this.submitted.emit(data);                    // 3. emit the captured value
  this.ngForm().resetForm();                    // 4. clear Material field touched/dirty state
}
```

Order matters: capture the value before resetting, and call `resetForm()` last.

### Cancel action
```typescript
cancel(): void {
  this.formGroup.reset();
  this.cancelled.emit();
}
```

The cancel method resets the form and emits `cancelled` so the parent can react (e.g. navigate away or close a dialog). It does not validate or submit.

---

## 6. Datepicker Integration

`NativeDateAdapter` must be provided at the **page component** level. It scopes the adapter to the entire component tree, including the child form component that hosts the `<input matDatepicker>`.

**Page component providers** (the routable page, not the form component):
```typescript
@Component({
  providers: [
    provideNativeDateAdapter(),
    { provide: MAT_DATE_LOCALE, useValue: 'es-MX' },
    FeatureFacade,
  ],
})
export default class FeaturePage { ... }
```

**Form component imports** (no providers — the adapter comes from the parent scope):
```typescript
@Component({
  imports: [
    ReactiveFormsModule,
    MatDatepicker,
    MatDatepickerInput,
    MatDatepickerToggle,
    MatSuffix,
    // ... other imports
  ],
})
export class FeatureFormComponent { ... }
```

**Form template**:
```html
<mat-form-field>
  <mat-label>Fecha</mat-label>
  <input matInput [matDatepicker]="picker" formControlName="date">
  <mat-datepicker #picker/>
  <mat-datepicker-toggle [for]="picker" matSuffix/>
  @if (formGroup.controls.date.invalid && formGroup.controls.date.touched) {
    <mat-error>La fecha es obligatoria</mat-error>
  }
</mat-form-field>
```

Do not add `provideNativeDateAdapter()` to the form component — it will be out of scope for the datepicker and cause a runtime error.

---

## 7. Anti-Patterns

- Do not emit values when the form is invalid.
- Do not read `formGroup.value` for non-nullable forms; use `getRawValue()`.
- Do not show validation errors before a field is touched, dirty, or after submit validation marks fields as touched.
- Do not put DTO conversion logic in the form component.
- Do not rely only on placeholder text as a label.
- Do not use `mat-flat-button`, `mat-raised-button`, or `color="primary"` — use `matButton="filled"` (Material 3 API).
- Do not add `provideNativeDateAdapter()` to the form component when it should be in the page component.

## 8. Final Checklist

- `ReactiveFormsModule` is in `imports`.
- Form is built with `NonNullableFormBuilder`.
- Submit (`submitted`) and cancel (`cancelled`) are both exposed with `output<T>()`.
- Submit marks invalid forms as touched and returns early.
- Material fields have labels and specific error messages.
- `#ngForm="ngForm"` is on the `<form>` element and `viewChild.required<FormGroupDirective>('ngForm')` is declared when full reset is needed.
- If the form has a datepicker, `provideNativeDateAdapter()` is in the page component's `providers`.
