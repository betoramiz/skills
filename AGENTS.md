# Agent Instructions

Use `.ai/skills/` as the shared project convention source.
Use `.ai/skills/angular/` as project-local context for the Angular skills.
Use `.ai/skills/dotnet/` as project-local context for the .NET skills (currently empty).

For Angular work, read the relevant `SKILL.md` files in `.ai/skills/angular` before implementing changes. Prefer the most specific skill for the task, starting with the architectural foundation:

- `.ai/skills/angular/architecture/SKILL.md` for feature structure, routes, layers, aliases, and layout conventions.
- `.ai/skills/angular/component/SKILL.md` for standalone components, signal inputs/outputs, template control flow, and component exports.
- `.ai/skills/angular/form-component/SKILL.md` for typed reactive forms, validators, and Material form layouts.
- `.ai/skills/angular/facade/SKILL.md` for feature state, signals, RxJS orchestration, and lifecycle cleanup.
- `.ai/skills/angular/routing/SKILL.md` for app-level layout routes, dashboard feature routes, lazy loading, and redirects.
- `.ai/skills/angular/service/SKILL.md` for REST services, `BaseService`, custom endpoints, and service placement.
- `.ai/skills/angular/material-m3-theming/SKILL.md` for Material 3 theme tokens, Sass mixins, and Tailwind coexistence.
- `.ai/skills/angular/testing/SKILL.md` for component, facade, service, form, and routing tests or regressions.

If the environment supports native skills, prefer invoking the matching `$angular-*` skill name. If not, treat the `SKILL.md` files above as required project context.
