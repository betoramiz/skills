---
name: angular-form-component
description: Build or refactor typed Angular reactive form components for this project. Use when creating form UI, NonNullableFormBuilder groups, Angular Material form fields, typed FormGroup models, form validation states, output submit events, or DTO mapping for modern Angular v19+ or v21+.
---

# Angular Form Components

Use typed reactive forms with `NonNullableFormBuilder`. Keep form UI components focused on validation and emitting values; map DTOs in feature models or facades.

## Core Rules

- Import `ReactiveFormsModule` in the component `imports`.
- Inject `NonNullableFormBuilder` with `inject()`.
- Build the form in a private helper and expose it to the template as `protected readonly`.
- Use `output<T>()` for submit/cancel events.
- Use `getRawValue()` for non-nullable typed values.
- Check `touched` or `dirty` before showing validation errors.
- Render `mat-error` messages for specific validation keys such as `required`, `email`, `minlength`, or domain validators.
- Add appropriate input attributes such as `type`, `autocomplete`, `min`, `max`, and `aria-describedby` when they improve browser and assistive behavior.
- Set `type="button"` on cancel buttons and `type="submit"` on save buttons.
- Put explicit form value/control types in `models/` for large or shared forms.
- For edit forms, initialize from a typed value and prefer `patchValue` only when the incoming object is intentionally partial.

```typescript
import { Component, inject, output } from '@angular/core';
import { NonNullableFormBuilder, ReactiveFormsModule, Validators } from '@angular/forms';
import { MatButton } from '@angular/material/button';
import { MatFormField, MatLabel, MatError } from '@angular/material/form-field';
import { MatInput } from '@angular/material/input';
import { UserFormGroup, UserFormValue } from '../../models/user-form-models';

@Component({
  selector: 'app-user-form',
  imports: [ReactiveFormsModule, MatButton, MatFormField, MatLabel, MatError, MatInput],
  templateUrl: './user-form.html',
})
export class UserFormComponent {
  private readonly fb = inject(NonNullableFormBuilder);

  protected readonly formGroup: UserFormGroup = this.buildForm();

  submitted = output<UserFormValue>();
  cancelled = output<void>();

  protected onSave(): void {
    if (this.formGroup.invalid) {
      this.formGroup.markAllAsTouched();
      return;
    }

    this.submitted.emit(this.formGroup.getRawValue());
  }

  private buildForm(): UserFormGroup {
    return this.fb.group({
      name: ['', Validators.required],
      email: ['', [Validators.required, Validators.email]],
    });
  }
}
```

Read [references/form-component-patterns.md](references/form-component-patterns.md) for simple vs. explicit typing, Material layout examples, and validation UI patterns.
