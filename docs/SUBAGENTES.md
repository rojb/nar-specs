# Subagentes

Investigación de RRG-73 (`add-subagents`), requisito de la materia. Documentación leída el
2026-11-04; estos productos cambian seguido, así que los detalles pueden quedar viejos. Este
documento resume lo que dicen sus páginas oficiales: no probamos Codex ni OpenCode, y el segmento
del curso (12:15:41) no se revisó para escribirlo.

## Qué son

Un subagente es otro ciclo de agente que el agente principal lanza para una tarea concreta. Tiene su
propio historial, su propio prompt y normalmente menos herramientas. Cuando termina, el principal
recibe solo su respuesta final.

```text
Agente principal   historial: pedido -> llamada a task -> resumen -> respuesta
      │ prompt                                 ▲ solo el resumen
      ▼                                        │
Subagente          historial propio: prompt -> 30 lecturas y búsquedas -> resumen
                   (se descarta al terminar)
```

## Por qué aíslan el contexto

Cada pedido reenvía el historial completo al modelo. Explorar un repo deja decenas de resultados
largos que solo sirven para llegar a una conclusión, y después ocupan la ventana de contexto, cuestan
tokens en cada llamada y compiten con lo importante. Si ese trabajo ocurre en un historial aparte, el
principal conserva la conclusión y no el camino.

Esto es distinto de la compactación (RRG-70). La compactación reduce un historial que ya creció; un
subagente evita que crezca.

**Lo que cuesta:**
- **Más tokens en total:** cada subagente hace sus propias llamadas. Codex lo advierte: consumen más
  tokens que hacer lo mismo con un solo agente. Se ahorra contexto, no necesariamente dinero.
- **Se pierden detalles:** si el resumen omite algo, el principal no lo tiene.
- **Empieza sin contexto:** el subagente no ve la conversación, así que el prompt debe decir todo lo
  que necesita.
- **Conviene más para leer que para escribir:** Codex los recomienda para trabajo de mucha lectura y
  pide más cuidado con escrituras en paralelo.

## Cómo lo resuelven

| | Claude Code | Codex | OpenCode |
|---|---|---|---|
| **Cómo se lanzan** | El modelo delega con la tool `Agent` | Por pedido directo o por una instrucción del proyecto o de un skill | El modelo con la tool `Task`, o el usuario con `@nombre` |
| **Tipos incluidos** | `Explore` y `Plan` (solo lectura), `general-purpose` y otros | `default`, `worker`, `explorer` | `general`, `explore` |
| **Solo lectura** | Herramientas de escritura denegadas | `explorer` es de mucha lectura, pero las instrucciones no crean un sandbox de solo lectura | `explore` es de solo lectura |
| **Subconjunto de herramientas** | `tools` y `disallowedTools` por agente | Puede desactivar capacidades, como `features.shell_tool` | Permisos por agente |
| **Aprobaciones** | Las pide en la sesión principal, nombrando al subagente en segundo plano | Hereda sandbox y aprobaciones del padre; sin interfaz, la acción falla y el error vuelve al padre | Permisos propios de cada agente |
| **Profundidad** | Hasta 3 niveles por defecto; configurable, y `1` lo apaga | Ver nota | Ver nota |
| **Límite de turnos** | `maxTurns`; el resultado vuelve marcado como parcial | No aparece en la página leída | `steps`; al llegar debe responder solo con texto |
| **Paralelo** | Sí, y en segundo plano (hasta 20 por defecto) | Sí | `general` puede ejecutar varias tareas en paralelo |
| **Qué ve el usuario** | Una fila por llamada; `/tasks` abre la transcripción | `/subagents` lista y cambia de hilo | Sesiones hijas por las que se puede navegar |
| **Definir los propios** | Markdown en `.claude/agents/` y `~/.claude/agents/` | TOML en `.codex/agents/` y `~/.codex/agents/` | Markdown en `.opencode/agents/`, `~/.config/opencode/agents/`, o `opencode.json` |

**Lo que no quedó claro en las fuentes:**
- **Codex y `max_depth`:** una versión anterior de su documentación daba `agents.max_depth` con valor
  1 por defecto, pero la página actual no lo menciona. No lo tomamos como vigente.
- **OpenCode y la profundidad:** la página principal no dice si un subagente puede lanzar otros. Otra
  página de su documentación (v2) dice que `general` no puede.
- **Claude Code y la profundidad:** varias guías de terceros dicen que los subagentes no pueden
  anidarse. La documentación oficial actual dice que sí, y seguimos a la oficial.

**Lo que tienen en común:**
- Historial nuevo y aislado; al padre vuelve un resultado, no el proceso.
- Un explorador de lectura separado de un agente general.
- Las aprobaciones no se relajan por estar dentro de un subagente.

## Qué decidimos implementar en nar

| Decisión | Por qué |
|---|---|
| Tool `task` que lanza un subagente con `ContextManager` propio y devuelve solo su respuesta final | Es el mecanismo común a los tres y lo pide el issue |
| Dos tipos fijos: `explorer` (`read_file`, `grep`, `glob`, `list_dir`) y `general` (las demás tools, con aprobaciones) | Los pide el issue, y coinciden con el patrón de los tres |
| El `explorer` no recibe ninguna tool de escritura | Quitarla del registro es una garantía; pedirle que no escriba no lo es, como advierte Codex |
| Sin anidar: `task` no está en el registro del subagente | Lo pide el issue. Quitar la tool es más simple que contar niveles |
| Límite de turnos configurable; al llegar, el resultado se marca como incompleto | El principal debe saber que el resumen puede estar a medias, como en Claude Code |
| El subagente usa la misma compuerta de aprobaciones y los mismos hooks | Un subagente no puede ser una forma de saltarse los controles |
| `general` no recibe `todos` | Esa lista es del agente principal; OpenCode tampoco se la da a `general` |
| Uno a la vez, en primer plano | Los pedidos de aprobación y la terminal son más simples. El paralelo queda fuera |
| Actividad plegada en una línea, sin guardar la transcripción | Lo pide el issue. Navegar por hilos exige una TUI que nar no tiene |
| Sin subagentes definidos por el usuario | Fuera del alcance del issue. Se puede sumar después |

## Fuentes

- Claude Code, *Create custom subagents*: https://code.claude.com/docs/en/sub-agents
- Codex, *Subagents*: https://learn.chatgpt.com/docs/agent-configuration/subagents.md (antes en
  https://developers.openai.com/codex/subagents)
- OpenCode, *Agents*: https://opencode.ai/docs/agents/
