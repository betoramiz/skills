---
name: angular-component
description: Build or refactor modern Angular components for this project. Use when creating standalone Angular v19+ or v21+ components, routable pages, nested UI components, signal inputs and outputs, template control flow, component imports, or project-specific Angular component file conventions.
---

# Angular Components

Follow the project's standalone Angular style. Prefer the local conventions over generic Angular CLI defaults.

## Core Rules

- Create standalone components only; declare all dependencies in `imports`.
- Use functional `inject()` for dependencies instead of constructor injection.
- Use signal-based `input()` and `output()` instead of `@Input()`, `@Output()`, and `EventEmitter`.
- Use native template control flow: `@if`, `@for`, `@switch`; avoid `*ngIf`, `*ngFor`, and `*ngSwitch`.
- Use the four granular aliases (`@shared-component/*`, `@shared-services/*`, `@shared-model/*`, `@shared-directives/*`) for shared imports; use relative imports within the same feature.
- Use separate `templateUrl` and `styleUrl` files for all feature components under `features/` and `components/`. Reserve inline `template`/`styles` for micro-components in `shared/components/layout/` (PageHeader, PageBody) or one-element dialog shells.
- Expose facade signals to templates with `.asReadonly()`: `protected entries = this.facade.entryList.asReadonly();`. Never expose writable signals to templates.
- Use `viewChild.required<T>('ref')` for template element or directive references instead of `@ViewChild()`. It returns a signal — call it to read: `this.ngForm()`. Use non-required `viewChild<T>()` when the element may be absent.

## File and Export Conventions

- Routable feature pages: `<feature>.ts`, `<feature>.html`, `<feature>.css`, and `export default class`.
- Nested feature components: `components/<name>/<name>.ts` and named exports.
- Layouts and shared dialogs: `<name>-component.ts`, matching the existing shared/layout style.

```typescript
import { Component, computed, input, output } from '@angular/core';
import { MatButton } from '@angular/material/button';

@Component({
  selector: 'app-action-panel',
  imports: [MatButton],
  template: `
    @if (visible()) {
      <section class="flex items-center justify-between gap-4">
        <h2>{{ title() }}</h2>
        <button matButton="filled" type="button" (click)="confirmed.emit()">
          {{ actionLabel() }}
        </button>
      </section>
    }
  `,
  styles: `
    section {
      min-height: 48px;
    }
  `,
})
export class ActionPanel {
  title = input.required<string>();
  visible = input<boolean>(true);
  confirmed = output<void>();

  protected readonly actionLabel = computed(() => `Save ${this.title()}`);
}
```

## Template Patterns

```html
@if (isLoading()) {
  <mat-spinner />
} @else if (errorMessage()) {
  <app-error [message]="errorMessage()" />
} @else {
  @for (item of items(); track item.id) {
    <app-item [item]="item" />
  } @empty {
    <p>No items found</p>
  }
}
```

Read [references/component-patterns.md](references/component-patterns.md) when creating a feature page, adding nested components, or checking project-specific naming and routing patterns.
