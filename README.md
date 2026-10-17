# nar-specs

Repositorio de planificación de **nar**, un agente de código para la terminal escrito en Python
(Click + Rich + prompt_toolkit). Proyecto de TAP 2026, 3er término, que replica el curso
["How Claude Code Works (By Building It)"](https://www.youtube.com/watch?v=3GjE_YAs03s) de Rivaan
Ranawat y suma los requisitos de la cátedra (compactación de contexto, sesiones, aprobaciones,
MCP externos, proveedores de modelo seleccionables, investigación de subagentes).

Aquí vive el store de OpenSpec (`nar-specs`) y la documentación. El plan completo está en
[`docs/ROADMAP.md`](docs/ROADMAP.md).

## Repos

| Repo | Contenido |
|---|---|
| `nar-specs` (este) | Store OpenSpec (`openspec/`), `docs/ROADMAP.md`, investigación |
| `nar` | Código Python. Se clona como carpeta anidada `nar/` dentro de este directorio (repo git independiente, ignorado por este) |

## Cómo trabajar un change

Regla: **1 issue de Linear = 1 change de OpenSpec = 1 owner**.

1. Tomá tu issue del proyecto [nar en Linear](https://linear.app/rgonaut/project/nar-e1a2c1e6a771)
   (verificá en el ROADMAP que sus dependencias estén mergeadas).
2. Creá el change: `openspec new change <nombre> --store nar-specs` (o desde esta carpeta).
3. Completá `proposal` → `specs` → `design` → `tasks` en este repo y abrí PR.
4. Con los artefactos aprobados, implementá en `nar/` siguiendo `tasks.md`.
5. Cerrá con `openspec archive` cuando el change esté hecho.

## Convenciones

- **Ramas:** usa el nombre de rama que sugiere Linear para el issue (botón "Copy git branch
  name"; por ejemplo `rubenjustiniano96/rrg-60-scaffold-cli-...`).
- **Commits:** Conventional Commits mencionando el id, por ejemplo
  `feat(cli): add interactive mode (RRG-60)`.
- Los artefactos de OpenSpec se escriben en inglés; la documentación del equipo, en español.
