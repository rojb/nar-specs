# Design

## Context

`ToolRegistry.execute` (`src/nar/tools/registry.py`) looks up the tool, validates the arguments, calls `tool.execute(...)` inside a `try`, and passes the result to `_finish`, which truncates the output. The `try` turns a `ToolError` or any other exception into an error result. It does not look at the value the tool returns.

When `execute` returns `None`, `_finish` reads `result.output` and raises `AttributeError` outside the `try`. The error reaches the top of the CLI as `unexpected failure (AttributeError)` and the session ends with exit code 1.

Every tool goes through this one method: built-in tools (`src/nar/tools/builtin.py`), custom tools (`src/nar/tools/custom.py`) and MCP tools (`src/nar/mcp/manager.py`) are all registered in the same registry. One check covers all of them.

## Goals / Non-Goals

**Goals:**
- A tool that returns something other than a `ToolResult` gives the model an error and the session goes on.
- The message names the tool and the type that came back, so the author of a custom tool can fix it.

**Non-Goals:**
- Checking the fields of a valid `ToolResult`.
- Changing how a raised exception is handled.

## Decisions

### 1. Layout
```
src/nar/tools/registry.py   MODIFIED  ToolRegistry.execute checks the returned value before _finish
tests/test_tools_registry.py MODIFIED  None, a str and a dict become error results; a real ToolResult is untouched
tests/test_tools_custom.py  MODIFIED  a custom tool that returns None gives an error and the next call still works
```

### 2. Check after the call, inside the same flow
After the `try` block and before `_finish`, `execute` tests `isinstance(result, ToolResult)`. If it is false, `result` becomes `ToolResult.error(f"{name} returned {type(result).__name__} instead of a ToolResult")` with the default failure kind `tool error`, because the tool misbehaved and the model cannot fix it, but should know the call failed. For `None` the message reads `<tool> returned NoneType instead of a ToolResult`.

### 3. No new config keys
There are no new keys in `~/.nar/config.toml` or `.nar/config.toml`.

## Risks / Trade-offs

- **A tool that returns a subclass of `ToolResult`** → `isinstance` accepts it, as it should.
- **The model sees an error for a call that may have had side effects** → unavoidable; the tool ran, and the error message says what went wrong.

## Migration Plan

None.

## Open Questions

None.
