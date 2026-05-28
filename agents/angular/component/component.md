-----
name: component
description: This skill provides guidelines and patterns for building components in modern Angular (v19+).
---

# Standalone Components

All new components must be standalone. Do not use legacy `NgModule` declarations.
Standalone is the default. Define dependencies explicitly in the `imports` array.

## Component Structure Pattern
```typescript
import { Component, inject } from '@angular/core';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-example',
  imports: [
    CommonModule
  ],
  templateUrl: './example.html',
  styleUrl: './example.css',
})
export class ExampleComponent {
}
```

## Single-File Components (SFC)

For small and medium-sized components, always prefer inline `template` and `styles` instead of separate `.html` and `.css` files.
This keeps all relevant context in one place, increases readability, and simplifies component maintenance.

```typescript
import { Component, inject } from '@angular/core';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-example',
  imports: [
    CommonModule
  ],
  template: `
    <div class="container">
      Example
    </div>
  `,
  styles: `
    .container {
      display: flex;
      justify-content: center;
      align-items: center;
      height: 100%;
      width: 100%;
    }
  `,
})
export class ExampleComponent {
}

## Modern Dependency Injection (DI)

Always use the modern `inject()` function from `@angular/core` instead of constructor-based dependency injection. It is cleaner, works perfectly in class property initialization, and improves type safety.

## Signal Inputs & Outputs (v17.2+)

Always use the modern Signal-based `input()` and `output()` APIs instead of the legacy `@Input()` and `@Output()` decorators.

### 1. Signal Inputs
Inputs defined with `input()` are read-only Signals. They automatically track updates and can be computed or used in effects easily.

```typescript
import { Component, input, computed } from '@angular/core';

@Component({
  selector: 'app-user-badge',
  template: `
    <span class="badge" [class.active]="isActive()">
      {{ displayName() }}
    </span>
  `
})
export class UserBadgeComponent {
  // Optional input with a default value
  isActive = input<boolean>(false);

  // Required input (no default value)
  firstName = input.required<string>();
  lastName = input.required<string>();

  // Computed signal derived from inputs
  displayName = computed(() => `${this.firstName()} ${this.lastName()}`);
}
```

### 2. Signal Outputs
Outputs are declared using the functional `output()` API instead of the legacy `@Output() / EventEmitter` decorators.

```typescript
import { Component, output } from '@angular/core';

@Component({
  selector: 'app-action-button',
  template: `<button (click)="notifyParent()">Click me</button>`
})
export class ActionButtonComponent {
  // Define modern output
  clicked = output<string>();

  notifyParent() {
    // Emit values cleanly
    this.clicked.emit('Button clicked!');
  }
}
```


## Template Syntax

Use native control flow—do NOT use `*ngIf`, `*ngFor`, `*ngSwitch`.

```html
<!-- Conditionals -->
@if (isLoading()) {
  <app-spinner />
} @else if (error()) {
  <app-error [message]="error()" />
} @else {
  <app-content [data]="data()" />
}

<!-- Loops -->
@for (item of items(); track item.id) {
  <app-item [item]="item" />
} @empty {
  <p>No items found</p>
}

<!-- Switch -->
@switch (status()) {
  @case ('pending') { <span>Pending</span> }
  @case ('active') { <span>Active</span> }
  @default { <span>Unknown</span> }
}


For detailed patterns, see [references/component-patterns.md](references/component-patterns.md).