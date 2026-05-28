# Pitaya AI Skills & Guidelines

Repositorio central de convenciones para agentes de codificacion usados en proyectos Pitaya.

La carpeta `.ai/` es la fuente portable: puede copiarse entre proyectos y ser referenciada por Codex, Claude Code, Gemini, Antigravity u otros agentes aunque cada herramienta tenga su propio mecanismo de descubrimiento.

## Estructura

```text
skills/
├── .ai/
│   ├── skills/
│   │   ├── angular/
│   │   │   ├── architecture/
│   │   │   │   ├── SKILL.md
│   │   │   │   └── references/
│   │   │   ├── component/
│   │   │   ├── facade/
│   │   │   ├── form-component/
│   │   │   ├── material-m3-theming/
│   │   │   ├── routing/
│   │   │   ├── service/
│   │   │   └── testing/
│   │   └── dotnet/
│   ├── prompts/
│   ├── templates/
│   └── agents/
│       ├── codex.md
│       ├── claude.md
│       ├── gemini.md
│       └── antigravity.md
├── AGENTS.md
├── CLAUDE.md
├── GEMINI.md
└── README.md
```

## Como Usarlo

Para agentes que leen instrucciones del repo, usa los archivos puente de la raiz:

- `AGENTS.md` para Codex y agentes compatibles.
- `CLAUDE.md` para Claude Code.
- `GEMINI.md` para Gemini.

Para Codex con skills globales, copia o sincroniza cada carpeta concreta que contiene un `SKILL.md` hacia:

```text
C:\Users\user-name\.codex\skills
```

Asi podras invocar skills como:

```text
Usa $angular-architecture y $angular-component.
```

Si una herramienta no reconoce skills nativas, referencia los archivos directamente:

```text
Lee @.ai/skills/angular/architecture/SKILL.md y @.ai/skills/angular/component/SKILL.md antes de editar.
```

## Skills Angular

- `.ai/skills/angular/architecture/` (`angular-architecture`): estructura de features, rutas, capas, aliases, Tailwind v4 y convenciones Material.
- `.ai/skills/angular/component/` (`angular-component`): componentes standalone, signals, control flow moderno y convenciones de archivos.
- `.ai/skills/angular/facade/` (`angular-facade`): estado local de feature, signals, RxJS y `takeUntilDestroyed`.
- `.ai/skills/angular/form-component/` (`angular-form-component`): formularios reactivos tipados con `NonNullableFormBuilder`.
- `.ai/skills/angular/material-m3-theming/` (`angular-material-m3-theming`): tokens Material 3, mixins Sass y coexistencia con Tailwind.
- `.ai/skills/angular/routing/` (`angular-routing`): rutas de layout, feature routes, redirects y `loadComponent`.
- `.ai/skills/angular/service/` (`angular-service`): servicios REST con `BaseService` y endpoints custom.
- `.ai/skills/angular/testing/` (`angular-testing`): pruebas de componentes, formularios, facades, servicios, rutas y regresiones.
