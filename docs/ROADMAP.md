# Roadmap de entrega — nar

**Equipo:** Ruben (Linear: Ruben Justiniano) y nathaliascode. **Regla:** 1 issue de Linear = 1
*change* de OpenSpec = 1 owner. Cada change pasa por proposal → specs → design → tasks en
`nar-specs` antes de escribir código en `nar`.

**Repos:** `nar-specs` (store OpenSpec + docs, este repo) y `nar` (código Python, carpeta anidada
`nar/`, repo git independiente). **Producto:** `nar`, agente de código CLI en Python (Click +
Rich, prompt_toolkit), réplica del curso *How Claude Code Works (By Building It)* de Rivaan
Ranawat (https://www.youtube.com/watch?v=3GjE_YAs03s) más los requisitos de la cátedra.

**Verificación:** cada change cierra con un checklist manual que ejecuta `nar` contra el criterio
"done when" del issue, más los chequeos adicionales que decida el owner.

**Tamaños:** S ≈ 1–2 días/persona, M ≈ 3–5 días, L ≈ 6–9 días. Estimación gruesa para comparar
carga relativa entre changes, no un compromiso de fechas.

**Linear:** equipo Rrgonaut (RRG), proyecto
[nar](https://linear.app/rgonaut/project/nar-e1a2c1e6a771).

---

## Fase 0 — Fundación

| Change | Issue | Owner | Contenido | Depende de | Tamaño |
|---|---|---|---|---|---|
| `scaffold-cli` | RRG-60 | Ruben | `pyproject` con entry point `nar = nar.cli:main` (`pip install -e .`), venv, ruff; CLI Click con modo interactivo (`nar`) y modo único (`nar "prompt"`); sistema de config `~/.nar/config.toml` + `.nar/config.toml` (provider/model, nivel de aprobación, servidores MCP); carga de `AGENTS.md`; manejo base de errores. Video: 00:01:28, 01:07:59, 06:29:22. | — | S |
| `add-llm-providers` | RRG-61 | nathaliascode | Interfaz común `LLMProvider` (stream/complete, formato interno de mensajes y tool calls); adaptadores Anthropic (SDK oficial), OpenRouter (compatible OpenAI) y Ollama (local, compatible OpenAI); selección por config y `--provider/--model`; streaming y no streaming; reintentos con backoff exponencial. Video: 00:05:20, 01:01:09. | RRG-60 | M |

## Fase 1 — Núcleo agéntico

| Change | Issue | Owner | Contenido | Depende de | Tamaño |
|---|---|---|---|---|---|
| `add-tool-framework` | RRG-62 | Ruben | Base `Tool` (name, description, parámetros JSON-schema, execute, `ToolResult`), `ToolRegistry` que exporta schemas por provider, tool `read_file`, render Rich de llamadas y errores, flag `requires_approval`. Video: 02:30:46, 03:09:17, 03:48:31, 10:03:11. | RRG-60 | M |
| `add-agent-loop` | RRG-63 | nathaliascode | Loop agéntico con límite de turnos, `ContextManager` (historial, conteo de tokens, system prompt + AGENTS.md), system prompt, modo único + REPL interactivo con salida Rich en streaming. Video: 01:15:05, 02:05:35, 04:34:55, 06:13:08, 06:29:22. | RRG-61, RRG-62 | M |

## Fase 2 — Herramientas

| Change | Issue | Owner | Contenido | Depende de | Tamaño |
|---|---|---|---|---|---|
| `add-file-tools` | RRG-64 | Ruben | `write_file`, `edit_file` (reemplazo exacto único, muestra diff), `list_dir` (ignora .git/node_modules/.venv). Video: 07:44:21, 08:25:08, 09:45:51. | RRG-62 | M |
| `add-shell-tool` | RRG-65 | Ruben | Tool de comandos con timeout, stdout/stderr/exit code, truncado, PowerShell en Windows + bash, blocklist, requiere aprobación. Video: 08:58:15. | RRG-62 | S |
| `add-search-tools` | RRG-66 | nathaliascode | `grep` (regex, formato file:line:text, tope de resultados) y `glob`; respetan `.gitignore`. Video: 10:06:07, 10:08:27, 10:37:19. | RRG-62 | S |
| `add-web-tools` | RRG-67 | nathaliascode | `web_search` (DuckDuckGo, sin API key) y `web_fetch` (URL → markdown, truncado, timeout). Video: 10:48:33, 11:03:18. | RRG-62 | S |
| `add-planning-memory-tools` | RRG-68 | nathaliascode | Tool `todos` renderizada en la TUI; tool `memory` persistida en `~/.nar/memory.json` e inyectada al inicio de sesión; actualización del system prompt. Video: 11:17:06, 11:39:20, 12:06:43. | RRG-63 | M |

## Fase 3 — Control y contexto

| Change | Issue | Owner | Contenido | Depende de | Tamaño |
|---|---|---|---|---|---|
| `add-approval-system` | RRG-69 | Ruben | Pregunta sí/no/sí-para-la-sesión antes de tools con `requires_approval`, mostrando comando/diff; niveles de autonomía `ask`, `auto-edit`, `yolo`; el motivo de rechazo vuelve al LLM. Video: 15:38:32. | RRG-63, RRG-64, RRG-65 | M |
| `add-context-compaction` | RRG-70 | nathaliascode | Poda de resultados viejos de tools primero, luego compactación por resumen con LLM; umbral configurable; `/compact`; indicador de uso de contexto. Video: 14:41:22, 15:24:12. | RRG-63 | M |
| `add-loop-detection` | RRG-71 | Ruben | Detecta llamadas idénticas repetidas y oscilación A-B; inyecta mensaje de cambio de estrategia y luego corta; bucle de feedback de errores estructurado. Video: 17:14:26. | RRG-63 | S |
| `add-hooks` | RRG-72 | Ruben | Eventos `before_tool`/`after_tool`/`before_agent`/`after_agent`/`on_error`; hooks de shell desde config que reciben JSON por stdin; `before_tool` puede bloquear; ejemplo: ruff tras `edit_file` sobre `.py`. Video: 16:32:00. | RRG-63 | M |

## Fase 4 — Extensibilidad

| Change | Issue | Owner | Contenido | Depende de | Tamaño |
|---|---|---|---|---|---|
| `add-subagents` | RRG-73 | nathaliascode | Documento de investigación `docs/SUBAGENTES.md` en `nar-specs` (fase explore); tool `task` que lanza un subagente con su propio `ContextManager` y subconjunto de tools y devuelve solo un resumen; tipos `explorer` (solo lectura) y `general`; límites de turnos y profundidad; render colapsado en la TUI. Video: 12:15:41. | RRG-63 | L |
| `add-mcp-client` | RRG-74 | Ruben | SDK oficial `mcp` de Python, stdio + streamable HTTP; servidores en config `[mcp_servers.<name>]`; tools registradas como `mcp__<server>__<tool>`; pasan por aprobaciones; `/mcp`; demo documentada con un servicio externo real (Jira/Atlassian, Teams o Linear). Video: 13:30:27. | RRG-62, RRG-69 | L |
| `add-custom-tools` | RRG-75 | nathaliascode | Carga de subclases de `Tool` desde `.nar/tools/` y `~/.nar/tools/`; los fallos avisan sin romper; tool de ejemplo + guía en README. Video: 13:02:55. | RRG-62 | S |

## Fase 5 — Sesiones y UX

| Change | Issue | Owner | Contenido | Depende de | Tamaño |
|---|---|---|---|---|---|
| `add-sessions` | RRG-76 | nathaliascode | Sesión con id/fecha/cwd/modelo/historial completo; autoguardado en `~/.nar/sessions/<id>.json` tras cada turno; `nar --resume [id]`, `/sessions`, `/resume`. Video: 07:32:00, 18:16:12, 18:18:27. | RRG-63 | M |
| `add-checkpoints` | RRG-77 | nathaliascode | `/checkpoint [name]`, `/checkpoints`, `/restore <id>`; documenta la diferencia sesión vs checkpoint. Video: 18:16:12, 18:55:01. | RRG-76 | S |
| `add-tui-commands` | RRG-78 | Ruben | Router de slash commands con autocompletado de prompt_toolkit; `/help /clear /exit /model /approval /config /tools /tokens`; registro abierto para que otros changes agreguen sus comandos. Video: 17:39:50. | RRG-63 | M |

---

## Paralelismo y secuencia

- **Arranque:** `add-llm-providers` y `add-tool-framework` dependen solo de `scaffold-cli`
  (RRG-60, de Ruben). En cuanto mergea, **Ruben hace RRG-62 mientras nathaliascode hace RRG-61 en
  paralelo**.
- **Primer punto de sincronía:** `add-agent-loop` (RRG-63) necesita ambos (RRG-61 + RRG-62). Hasta
  que no mergea, la mayor parte de las Fases 3 a 5 no puede empezar.
- **Herramientas (Fase 2):** las tools solo requieren el framework (RRG-62), así que las tools de
  archivos y shell de Ruben (RRG-64, RRG-65) y las de búsqueda y web de nathaliascode (RRG-66,
  RRG-67) corren en paralelo con, o después de, RRG-63. `add-planning-memory-tools` (RRG-68)
  sí espera al loop.
- **Tras RRG-63:** RRG-68, RRG-70, RRG-71, RRG-72, RRG-73, RRG-76 y RRG-78 quedan desbloqueados y
  pueden repartirse entre ambos según la carga.
- **Cadenas con dependencia cruzada:** `add-approval-system` (RRG-69) espera a RRG-64 y RRG-65;
  `add-mcp-client` (RRG-74) espera a RRG-69; `add-checkpoints` (RRG-77) espera a RRG-76.

## Cobertura de requisitos de la cátedra

| Requisito | Cubierto por |
|---|---|
| Gestión de contexto (compactación + poda) | RRG-63 (`ContextManager`) + RRG-70 |
| Sesiones con historial | RRG-76 (+ RRG-77 checkpoints) |
| Human in the loop (aprobaciones) | RRG-69 |
| Conexión a servidores MCP externos (Jira, Teams, Linear) | RRG-74 |
| Proveedor de modelo seleccionable (Anthropic, OpenRouter, Ollama) | RRG-61 (+ `/model` en RRG-78) |
| Investigación sobre subagentes | RRG-73 (`docs/SUBAGENTES.md`) |
| Uso de OpenSpec | este repo, todos los changes |

## Balance de carga

| Persona | S | M | L | Changes |
|---|---|---|---|---|
| Ruben | 3 | 5 | 1 | RRG-60, 65, 71 (S); 62, 64, 69, 72, 78 (M); 74 (L) |
| nathaliascode | 4 | 5 | 1 | RRG-66, 67, 75, 77 (S); 61, 63, 68, 70, 76 (M); 73 (L) |

Total: 19 changes. Carga equilibrada dentro de la estimación gruesa (la única diferencia es una
S más para nathaliascode).
