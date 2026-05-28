# Gemini Instructions

Use `.ai/skills/` as the shared project convention source.

For Angular work, read the relevant `SKILL.md` files in `.ai/skills/` before implementing changes. Prefer the most specific skill for the task, starting with the architectural foundation:

- **Architecture**: `.ai/skills/angular/architecture/SKILL.md` (Core conventions, layers flow, layouts)
- **Components**: `.ai/skills/angular/component/SKILL.md` (Standalone, signal inputs/outputs, control flow)
- **Forms**: `.ai/skills/angular/form-component/SKILL.md` (Typed reactive forms, validators, Material forms)
- **Facades**: `.ai/skills/angular/facade/SKILL.md` (Feature state, signals, RxJS orchestration)
- **Routing**: `.ai/skills/angular/routing/SKILL.md` (Layout routing, dashboard feature routes, lazy loading)
- **Services**: `.ai/skills/angular/service/SKILL.md` (REST API wrappers, `BaseService`, HTTP calls)
- **Material Theming**: `.ai/skills/angular/material-m3-theming/SKILL.md` (Material 3 overrides, SCSS, custom theme tokens)
- **Testing**: `.ai/skills/angular/testing/SKILL.md` (Unit tests, facades, components, services, and forms)
