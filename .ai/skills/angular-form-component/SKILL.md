---
name: angular-form-component
description: Build or refactor typed Angular reactive form components for this project. Use when creating form UI, NonNullableFormBuilder groups, Angular Material form fields, typed FormGroup models, form validation states, output submit/cancel events, form reset with Material field state, datepicker integration, or DTO mapping for modern Angular v19+ or v21+.
---

# Angular Form Components

Use typed reactive forms with `NonNullableFormBuilder`. Keep form UI components focused on validation and emitting values; map DTOs in feature models or facades.

## Core Rules

- Import `ReactiveFormsModule` in the component `imports`.
- Inject `NonNullableFormBuilder` with `inject()`.
- Build the form in a private helper and expose it to the template as `protected readonly`.
- Use `submitted = output<T>()` for the submit event and `cancelled = output<void>()` for the cancel event.
- Use `getRawValue()` for non-nullable typed values.
- Check `touched` or `dirty` before showing validation errors.
- Render `mat-error` messages for specific validation keys such as `required`, `email`, `minlength`, or domain validators.
- Add appropriate input attributes such as `type`, `autocomplete`, `min`, `max`, and `aria-describedby` when they improve browser and assistive behavior.
- Set `type="button"` on cancel buttons and `type="submit"` on save buttons.
- Put explicit form value/control types in `models/` for large or shared forms.
- For edit forms, initialize from a typed value and prefer `patchValue` only when the incoming object is intentionally partial.
- For full form reset (values + Material field state), declare `protected readonly ngForm = viewChild.required<FormGroupDirective>('ngForm')` and add `#ngForm="ngForm"` to the `<form>` element. Call `ngForm().resetForm()` after emitting.
- When the form contains a `MatDatepicker`, declare `provideNativeDateAdapter()` and `MAT_DATE_LOCALE` in the **page component's** providers, not the form component. The form component only imports the datepicker modules.

```typescript
import { Component, inject, output, viewChild } from '@angular/core';
import { FormGroupDirective, NonNullableFormBuilder, ReactiveFormsModule, Validators } from '@angular/forms';
import { MatButton } from '@angular/material/button';
import { MatFormField, MatLabel, MatError } from '@angular/material/form-field';
import { MatInput } from '@angular/material/input';
import { UserFormValue } from '../../models/user-form-models';

@Component({
  selector: 'user-form',
  imports: [ReactiveFormsModule, MatButton, MatFormField, MatLabel, MatError, MatInput],
  templateUrl: './user-form.html',
  styleUrl: './user-form.css',
})
export class UserFormComponent {
  private readonly fb = inject(NonNullableFormBuilder);

  protected readonly formGroup = this.buildForm();
  protected readonly ngForm = viewChild.required<FormGroupDirective>('ngForm');

  submitted = output<UserFormValue>();
  cancelled = output<void>();

  protected onSave(): void {
    if (this.formGroup.invalid) {
      this.formGroup.markAllAsTouched();
      return;
    }
    const data = this.formGroup.getRawValue();
    this.formGroup.reset();
    this.submitted.emit(data);
    this.ngForm().resetForm();  // clears Material field touched/dirty state
  }

  protected cancel(): void {
    this.formGroup.reset();
    this.cancelled.emit();
  }

  private buildForm() {
    return this.fb.group({
      name: ['', Validators.required],
      email: ['', [Validators.required, Validators.email]],
    });
  }
}
```

Template — include `#ngForm="ngForm"` on the form element:

```html
<form [formGroup]="formGroup" #ngForm="ngForm" (ngSubmit)="onSave()" class="flex flex-col gap-4 p-6">
  <mat-form-field class="w-full">
    <mat-label>Nombre</mat-label>
    <input matInput formControlName="name" autocomplete="name">
    @if (formGroup.controls.name.invalid && formGroup.controls.name.touched) {
      <mat-error>El nombre es obligatorio</mat-error>
    }
  </mat-form-field>

  <div class="flex justify-end gap-2.5">
    <button matButton="text" type="button" (click)="cancel()">Cancelar</button>
    <button matButton="filled" type="submit" [disabled]="formGroup.invalid">Guardar</button>
  </div>
</form>
```

Read [references/form-component-patterns.md](references/form-component-patterns.md) for simple vs. explicit typing, Material layout examples, form reset patterns, datepicker integration, and validation UI patterns.
