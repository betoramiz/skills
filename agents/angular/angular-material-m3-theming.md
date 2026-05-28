# Angular Material 3 Customization and Theme Styling

This skill provides guidelines and patterns for styling and customizing Angular Material 3 (M3) components using modern Sass (SCSS) mixins and integrating them seamlessly with Tailwind CSS.

---

## Theming Architecture

Angular Material 3 uses CSS custom properties (variables) defined under the M3 design specification.
Customizations are managed globally via a dedicated SCSS theme file (`src/themes/custom-theme.scss`) using standard `@use '@angular/material' as mat;`.

### Basic Theme Definition
```scss
@use '@angular/material' as mat;
@use 'colors.css';

html {
  // Apply the typography and base systems
  @include mat.theme((
    typography: Roboto
  ));

  // Set the default color scheme (light / dark)
  color-scheme: light;

  // Use Angular Material's system-level CSS variables
  background-color: var(--mat-sys-surface);
  color: var(--mat-sys-on-surface);
  font: var(--mat-sys-body-medium);
}
```

---

## Component Customization (Overrides)

Always use modern component-specific override mixins provided by Angular Material instead of writing arbitrary CSS selectors.

### Override Patterns:
Use typed maps in the override mixins:

```scss
// Sidenav custom styling
.side-nav-flat {
  @include mat.sidenav-overrides((
    container-width: 250px,
    container-shape: square,
    container-background-color: var(--mat-sys-on-primary)
  ));
}

// Navigation list overrides
@include mat.list-overrides((
  active-indicator-color: var(--mat-sys-primary),
  active-indicator-shape: 0,
));

// Button border-radius (shape) overrides
@include mat.button-overrides((
  filled-container-shape: 8px,
  text-container-shape: 8px,
));
```

---

## Coexistence with Tailwind CSS v4

Combine Tailwind CSS utility classes with customized Angular Material components to build highly premium interfaces.

### Best Practices:
1. **Layout & Spacing**: Use Tailwind utilities for positioning, flexing, margins, and padding.
2. **Colors**: Use Angular Material M3 system variables (`var(--mat-sys-...)`) or Tailwind color classes when appropriate to ensure theme consistency.
3. **Styling Components**: Avoid inline overriding of Material classes in component HTML. Customize components globally inside SCSS or by passing semantic modifier classes (e.g. `button.success` using custom overrides).
