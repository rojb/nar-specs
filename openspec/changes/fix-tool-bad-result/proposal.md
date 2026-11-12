# Proposal

- **Owner:** Ruben
- **Linear issue:** RRG-95
- **Video timestamps covered:** none (found by the real-API end-to-end run on 2026-11-12)
- **Depends on:** none (builds on `custom-tools`, merged)
- **Course requirement covered:** none of the listed ones; it is a correctness fix on the tool framework

## Why

The `tool-framework` spec says a failure inside a tool becomes an error result and never terminates `nar`. A tool whose `execute` returns `None`, or anything else that is not a `ToolResult`, breaks that promise. The real-API run of 2026-11-12 showed `error: unexpected failure (AttributeError): 'NoneType' object has no attribute 'output'` and exit code 1. One careless custom tool ends the session.

## What Changes

- `ToolRegistry.execute` checks what a tool returned. A value that is not a `ToolResult` becomes an error result: `<tool> returned <type name> instead of a ToolResult`.
- The model receives that error on that call and the session goes on.
- The check applies to every tool in the registry: built-in, custom (`.nar/tools/`) and MCP.

Out of scope: validating the fields inside a `ToolResult`, and changing how a tool that raises is handled (that already works).

**"Done when" (RRG-95):** a custom tool in `.nar/tools/` that returns `None` shows an error on that call, and `nar` keeps answering.

## Capabilities

### New Capabilities

None.

### Modified Capabilities

None. The `tool-framework` requirement on tool results already says a failure inside a tool SHALL become an error result and SHALL NOT terminate `nar`. This change makes the code do it, so there is no spec delta (`skip_specs: true`).

## Impact

- **Code (`nar` repo):** `src/nar/tools/registry.py` (`ToolRegistry.execute`), tests.
- **Behavior:** a tool that returned `None` used to end the session and now gives the model an error. Correct tools are not affected.
- **Dependencies:** none.
