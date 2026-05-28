# 🛠️ Pitaya Agent Skills & Guidelines

¡Bienvenido al repositorio central de **Pitaya Skills**! Este proyecto funciona como el "cerebro de directrices" para asistentes de codificación con Inteligencia Artificial (tales como Gemini, Claude, GitHub Copilot, Cursor y custom GPTs) dentro del ecosistema de desarrollo de **Pitaya**.

El objetivo principal de este proyecto es estandarizar la arquitectura de software, las convenciones de diseño, los patrones de código y las directrices tecnológicas, asegurando que cualquier código generado por IA sea coherente, moderno y de calidad de producción.

---

## 🎯 Objetivo del Proyecto

En el desarrollo de software moderno, los asistentes de código de IA son miembros activos del equipo. Sin embargo, para que sus contribuciones sean efectivas, deben alinearse con las decisiones técnicas particulares del proyecto. 

Este repositorio resuelve ese reto al:
1. **Definir "Habilidades" (Skills) de IA:** Archivos Markdown altamente estructurados con metadatos descriptivos que explican con precisión cómo resolver problemas específicos (por ejemplo: cómo estructurar un componente Angular, cómo implementar una fachada, o cómo gestionar temas con Angular Material 3).
2. **Establecer un Contexto Unificado:** Servir como una fuente de verdad única para el estilo de código del frontend y backend de Pitaya.
3. **Facilitar la Inyección de Contexto:** Permitir que los desarrolladores alimenten fácilmente a la IA con estas reglas de forma directa o mediante indexación automática en IDEs modernos (como Cursor o VS Code).

---

## 📂 Estructura del Repositorio

La arquitectura del proyecto está organizada de forma lógica por tecnologías y herramientas:

```text
skills/
├── agents/                       # Directrices técnicas específicas por plataforma
│   ├── angular/                  # Ecosistema Angular 19+
│   │   ├── architecture/         # Estructura del proyecto, flujo de datos y nomenclatura
│   │   ├── component/            # Reglas para componentes Standalone, SFC, Signals e inyección
│   │   ├── facade/               # Gestión de estado local y flujo RxJS reactivo
│   │   ├── form-component/       # Patrones de formularios reactivos y validaciones
│   │   ├── service/              # Capa REST y consumo de APIs de backend
│   │   └── angular-material-m3-theming.md  # Guía de diseño con Angular Material 3 y Tailwind CSS v4
│   └── dotnet/                   # Ecosistema .NET / C# (Próxima expansión)
├── prompts/                      # Prompts del sistema estandarizados para diferentes tareas
├── templates/                    # Plantillas de código y boilerplates listos para usar
├── claude.md                     # Entrada de contexto optimizada para Claude (Anthropic)
├── gemini.md                     # Entrada de contexto optimizada para Gemini (Google)
└── README.md                     # Este archivo informativo
```

---

## 🚀 Habilidades Destacadas (Angular 19+)

El conjunto de habilidades actual se enfoca en el desarrollo moderno de **Angular v19+** y sus mejores prácticas:

*   **Arquitectura en 3 Capas:** Separación estricta de responsabilidades entre la Vista (Componentes reactivos), el Controlador (Fachadas con Signals y RxJS) y los Datos (Servicios HTTP).
*   **Signals por Defecto:** Uso obligatorio de la API funcional de Signals para entradas y salidas (`input()`, `output()`, `computed`, `effect`) en lugar de decoradores heredados.
*   **Componentes Standalone:** Eliminación de los módulos heredados (`NgModule`) en favor de componentes auto-contenidos.
*   **Componentes de Archivo Único (SFC):** Preferencia por templates y estilos en línea para componentes medianos y pequeños.
*   **Coexistencia UI:** Integración perfecta de la biblioteca de diseño **Angular Material 3** (usando mixins de Sass personalizados) con el poder de **Tailwind CSS v4** para layouts flexibles.

---

## 💻 ¿Cómo Usar Este Proyecto?

Existen múltiples maneras de utilizar estas habilidades en tu flujo de trabajo diario con IA:

### 1. En IDEs Modernos (Cursor, VS Code + Copilot)
Puedes referenciar directamente las carpetas o archivos de habilidades usando comandos de contexto:
*   En Cursor, escribe `@architecture.md` o `@component.md` al hacer una consulta para asegurar que el código generado use la estructura y APIs correctas.
*   Puedes indexar toda la carpeta `agents/angular/` como contexto de referencia del proyecto.

### 2. Copiar como Contexto Inicial
Si usas interfaces web como Claude.ai o Gemini Advanced, puedes copiar el contenido del archivo específico de la habilidad que vas a desarrollar y pegarlo al inicio de tu prompt:
> *"Actúa como un desarrollador experto en Angular. A continuación te comparto las directrices de arquitectura de mi proyecto que debes seguir estrictamente: [pegar contenido de architecture.md]"*

---
