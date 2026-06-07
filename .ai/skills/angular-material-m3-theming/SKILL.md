---
name: angular-material-m3-theming
description: Customize Angular Material 3 theming for this project. Use when editing Material M3 theme SCSS, component override mixins, Material design tokens, Tailwind v4 coexistence, src/themes/custom-theme.scss, or visual styling of Angular Material components.
---

# Angular Material 3 Theming

Use Angular Material 3 tokens and override mixins instead of targeting internal Material DOM classes.

## Core Rules

- Keep global theme configuration in `src/themes/custom-theme.scss`.
- Use `@use '@angular/material' as mat;`.
- Use Material system CSS variables such as `var(--mat-sys-primary)` and `var(--mat-sys-surface)`.
- Use component override mixins such as `mat.button-overrides`, `mat.sidenav-overrides`, and `mat.list-overrides`.
- Use `mat.form-field-overrides`, `mat.dialog-overrides`, and related component mixins when changing Material form or dialog appearance.
- Use Tailwind utilities for layout and spacing; use Material tokens for component color, shape, and state styling.
- Prefer semantic modifier classes around Material components over deep selectors into generated Material classes.
- Avoid `::ng-deep` and selectors that depend on private MDC/Material DOM structure.

```scss
@use '@angular/material' as mat;

html {
  @include mat.theme((
    typography: Roboto,
  ));

  color-scheme: light;
  background-color: var(--mat-sys-surface);
  color: var(--mat-sys-on-surface);
  font: var(--mat-sys-body-medium);
}

.side-nav-flat {
  @include mat.sidenav-overrides((
    container-width: 250px,
    container-shape: 0,
    container-background-color: var(--mat-sys-surface-container),
  ));
}

.primary-actions {
  @include mat.button-overrides((
    filled-container-shape: 8px,
    text-container-shape: 8px,
  ));
}

.dense-form {
  @include mat.form-field-overrides((
    container-height: 48px,
    filled-container-shape: 8px,
    focus-active-indicator-color: var(--mat-sys-primary),
  ));
}

.confirmation-dialog {
  @include mat.dialog-overrides((
    container-shape: 8px,
    container-color: var(--mat-sys-surface-container-high),
  ));
}
```

When changing UI, verify the result in the browser at both desktop and mobile widths.

## Material 3 Button Variants

Angular Material 3 uses an attribute-based API — not directive selectors or `color=""` props.

| Variant    | Attribute            | Import          |
|------------|----------------------|-----------------|
| Filled     | `matButton="filled"` | `MatButton`     |
| Text       | `matButton="text"`   | `MatButton`     |
| Outlined   | `matButton="outlined"`| `MatButton`    |
| Icon only  | `matIconButton`      | `MatIconButton` |

```html
<button matButton="filled" type="submit">Guardar</button>
<button matButton="text" type="button" (click)="cancel()">Cancelar</button>
<button matIconButton type="button"><mat-icon>delete</mat-icon></button>
```

Never use: `mat-flat-button`, `mat-raised-button`, `mat-stroked-button`, `mat-button` (old directive), or `color="primary"`.

## Global Material Config (app.config.ts)

`MAT_FORM_FIELD_DEFAULT_OPTIONS` sets `appearance: 'outline'` for all form fields project-wide.
Do not set `appearance=""` on individual `<mat-form-field>` elements.

For datepickers, provide the adapter at the **page component** level (not `app.config.ts` and not the form component):

```typescript
@Component({
  providers: [
    provideNativeDateAdapter(),
    { provide: MAT_DATE_LOCALE, useValue: 'es-MX' },
  ],
})
export default class FeaturePage { ... }
```

`MAT_DATE_LOCALE` is **not** set globally — only in the page component that hosts a datepicker form.

## Anti-Patterns

- Do not style generated `.mat-mdc-*` internals from component CSS.
- Do not create one-off color values when a Material system token exists.
- Do not put global Material override mixins in feature component styles.
- Do not use `::ng-deep` for new theming work.

## Final Checklist

- Theme changes live in `src/themes/custom-theme.scss`.
- Overrides use Material mixins and system tokens.
- Layout spacing remains in Tailwind utilities where practical.
- UI is checked at desktop and mobile widths.
