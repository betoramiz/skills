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

Use semantic HTML tags and leverage standard Angular Material component forms inside form templates:

```html
<form [formGroup]="formGroup" (ngSubmit)="onSave()" class="flex flex-col gap-4 p-6">
  <!-- Name Input -->
  <mat-form-field appearance="fill" class="w-full">
    <mat-label>Nombre</mat-label>
    <input matInput formControlName="name" placeholder="Ej. Juan Pérez">
    @if (formGroup.controls.name.invalid && formGroup.controls.name.touched) {
      <mat-error>El nombre es obligatorio</mat-error>
    }
  </mat-form-field>

  <!-- Email Input -->
  <mat-form-field appearance="fill" class="w-full">
    <mat-label>Correo Electrónico</mat-label>
    <input matInput formControlName="email" type="email" placeholder="ejemplo@correo.com">
    @if (formGroup.controls.email.invalid && formGroup.controls.email.touched) {
      <mat-error>Introduce un correo válido</mat-error>
    }
  </mat-form-field>

  <!-- Actions -->
  <div class="flex justify-end gap-3 mt-4">
    <button mat-button type="button" (click)="onCancel()">Cancelar</button>
    <button mat-flat-button color="primary" type="submit" [disabled]="formGroup.invalid">Guardar</button>
  </div>
</form>
```

---

## 3. Form Validation and Styling Guidelines

- **Form States**: Ensure validation triggers are responsive. Check validation with `formGroup.controls.controlName.touched` or `formGroup.controls.controlName.dirty` before rendering errors to prevent early red markings on pristine fields.
- **Specific Errors**: Prefer `hasError('required')`, `hasError('email')`, and custom validation keys over a generic `invalid` message when multiple failures are possible.
- **Browser Semantics**: Use native attributes such as `type="email"`, `autocomplete="email"`, `min`, `max`, and `maxlength` when they match the domain.
- **Tailwind Grid Alignment**: Arrange form fields in grid structures (`grid grid-cols-1 md:grid-cols-2 gap-4`) for spacious layouts.
- **Button Styling**: Always set `type="button"` on the Cancel button to prevent it from accidentally submitting the HTML form. Set `type="submit"` on the primary save action.

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

## 5. Anti-Patterns

- Do not emit values when the form is invalid.
- Do not read `formGroup.value` for non-nullable forms; use `getRawValue()`.
- Do not show validation errors before a field is touched, dirty, or after submit validation marks fields as touched.
- Do not put DTO conversion logic in the form component.
- Do not rely only on placeholder text as a label.

## 6. Final Checklist

- `ReactiveFormsModule` is in `imports`.
- Form is built with `NonNullableFormBuilder`.
- Submit and cancel are exposed with `output<T>()`.
- Submit marks invalid forms as touched and returns early.
- Material fields have labels and specific error messages.
