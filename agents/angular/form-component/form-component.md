-----
name: form-component
description: This skill provides guidelines and patterns for building form-type components in modern Angular (v19+).
---

# Angular Modern Form Components

This skill provides guidelines and patterns for creating reactive form components in modern Angular (v19+ and v21+) using `NonNullableFormBuilder`, strictly-typed form groups, and clean separation of concerns.

---

## 1. Strictly-Typed Form Models

Always define interfaces for form values and controls inside a feature-specific `models/` directory. This guarantees full type safety and prevents runtime value reference errors.

### Pattern:
```typescript
import { FormControl, FormGroup } from '@angular/forms';

export interface UserFormValue {
  name: string;
  email: string;
}

export interface UserFormControls {
  name: FormControl<string>;
  email: FormControl<string>;
}

// Explicit FormGroup type mapping
export type UserFormGroup = FormGroup<UserFormControls>;
```

---

## 2. Injecting NonNullableFormBuilder

Always use `NonNullableFormBuilder` to automatically enforce that form values cannot be null. Resolve it using the functional `inject()` API.

```typescript
import { Component, inject } from '@angular/core';
import { NonNullableFormBuilder } from '@angular/forms';

export class UserFormComponent {
  private readonly fb = inject(NonNullableFormBuilder);
}
```

---

## 3. Designing the Form Component

Define the form group inside a private helper method and assign it to a `protected readonly` property. This makes it visible to the template while protecting it from external mutations.

```typescript
import { Component, inject, output } from '@angular/core';
import { NonNullableFormBuilder, Validators, ReactiveFormsModule } from '@angular/forms';
import { UserFormValue, UserFormGroup } from '../../models/form-models';

@Component({
  selector: 'app-user-form',
  imports: [ReactiveFormsModule],
  templateUrl: './user-form.html',
})
export class UserFormComponent {
  private readonly fb = inject(NonNullableFormBuilder);

  // Protected to allow template access, readonly to prevent external mutations
  protected readonly formGroup: UserFormGroup = this.buildForm();
  
  // Modern output emitter
  onSubmit = output<UserFormValue>();

  protected onSave() {
    if (this.formGroup.invalid) {
      return;
    }
    // Safely emit typed non-nullable values
    this.onSubmit.emit(this.formGroup.getRawValue());
  }

  protected onCancel() {
    this.formGroup.reset();
  }

  private buildForm(): UserFormGroup {
    return this.fb.group({
      name: ['', Validators.required],
      email: ['', [Validators.required, Validators.email]],
    });
  }
}
```

For detailed architectural form layouts, validation UI, and verbose/simple typing patterns, see [references/form-component-patterns.md](references/form-component-patterns.md).
