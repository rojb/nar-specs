# Proposal

- **Owner:** Ruben
- **Linear issue:** RRG-74
- **Video timestamps covered:** 13:30:27 (MCP)
- **Depends on:** RRG-62 `add-tool-framework` (archived; `Tool`, `ToolRegistry`) and RRG-69 `add-approval-system` (merged in `nar`, change not archived yet; MCP tools take part in approval through the generic `Tool.preview` / `requires_approval` hooks it introduced). **Archive order:** none required. Every delta in this change is an ADDED requirement on a capability that exists today (`configuration`, `cli-entrypoint`) or a new capability (`mcp-client`); nothing modifies a requirement introduced by an unarchived change. The new `mcp-client` spec refers to the `approval-system` capability by name, which only exists once `add-approval-system` is archived; archive that change first so the reference resolves.
- **Unblocks:** RRG-78 `add-tui-commands` (absorbs the minimal slash-command seam added here, see Decision 9 in `design.md`), RRG-73 `add-subagents` (can be given a registry with or without MCP tools).
- **Course requirement covered:** connecting external MCP servers (Jira, Microsoft Teams, Linear).

## Why

The course asks that `nar` can use tools it does not ship with, provided by external services through the Model Context Protocol. Today `mcp_servers` exists in the configuration only as a reserved, untyped table that nothing reads. With the tool framework (RRG-62), the agent loop (RRG-63) and the approval system (RRG-69) in place, an MCP server's tools can be exposed to the model exactly like built-in tools, and every call is gated by the human in the loop. The team tracks its work in Linear, so Linear's official MCP server is the real external service for the documented demo.

## What Changes

- **MCP client** built on the official Python SDK (`mcp`), with two transports:
  - **stdio:** `nar` starts the server as a child process (`command`, `args`, `env`) and talks over its standard input and output.
  - **streamable HTTP:** `nar` connects to a server `url` with optional HTTP `headers`.
- **Server definitions** in `~/.nar/config.toml` / `.nar/config.toml` as `[mcp_servers.<name>]` with `command`, `args`, `env` (stdio) or `url`, `headers` (HTTP), plus `enabled`, `enabled_tools`, `startup_timeout` and `tool_timeout`. The untyped table becomes a validated one: exactly one of `command` or `url`, unknown keys rejected.
- **Secrets stay out of files:** `${VAR}` references in `command`, `args`, `env`, `url` and `headers` are replaced from the environment when the server is started, so `headers = { Authorization = "Bearer ${LINEAR_API_KEY}" }` is safe to commit. `nar --show-config` masks literal secrets in `env` and `headers` values and shows `${VAR}` references as written.
- **Discovery at startup:** every enabled server is connected (concurrently, each with a startup timeout), its tools are listed, and each one is registered in the `ToolRegistry` as `mcp__<server>__<tool>`. Names are sanitized to what every provider accepts (letters, digits, underscore; at most 64 characters) and collisions are never silent.
- **Tools behave like built-in tools:** the MCP input schema is passed through as the tool's parameters, calls are validated by the registry, the result's text content is returned to the model (non-text content is described, never dumped), a tool error comes back as an error result, a call has a timeout, and the existing output limit applies.
- **Approval:** every MCP tool requires approval (nothing in the approval code lists MCP tool names). The prompt names the server and the original tool and shows the arguments. Servers' own "read-only" hints are not trusted.
- **A server that cannot start never breaks `nar`:** a missing command, a connection error, a timeout, a missing `${VAR}` or a rejected token produce one warning naming the server and the reason, and the other servers and the built-in tools keep working. A server that dies later turns its tools into error results.
- **`/mcp` command** in interactive mode lists servers, their status (connected / failed / disabled) and their tools; `/mcp <server>` narrows it to one server. It is added through a minimal slash-command table (`/`-prefixed lines are commands, not requests) that RRG-78 replaces with its router.
- **Clean shutdown:** servers are stopped when `nar` exits (including after Ctrl+C), and a child process never outlives `nar`.
- **Documented demo with Linear's official remote MCP server** (`https://mcp.linear.app/mcp`), authenticated with a Linear personal API key sent as `Authorization: Bearer`, plus a fallback through `npx mcp-remote` over stdio. README section "MCP servers" with both configurations.

Out of scope: MCP resources, prompts, sampling, elicitation and roots; OAuth inside `nar` (the demo uses an API key; see `design.md` Decision 6); reconnecting a server that died or refreshing the tool list while running (restart `nar`); a per-tool approval policy or allow-list; the `/approval`, `/tools` and other slash commands and prompt completion (RRG-78); a `.env` file loader; running `nar` itself as an MCP server.

**"Done when" (RRG-74):** `nar` lists the issues of a real project through an external MCP server.

## Capabilities

### New Capabilities

- `mcp-client`: connecting configured MCP servers over stdio and streamable HTTP, secret references from the environment, discovery and registration of tools as `mcp__<server>__<tool>`, tool names and schemas, calling a tool and mapping its result, approval of MCP tools, failure isolation, timeouts, cancellation, shutdown and the `/mcp` command.

### Modified Capabilities

- `configuration`: new requirements for the structure of an `mcp_servers` entry and for masking secrets in `--show-config`. These are ADDED requirements next to the existing "Initial settings" requirement, which already lists `mcp_servers`; no archive-order dependency.
- `cli-entrypoint`: new requirement that, in interactive mode, a line starting with `/` is a command and not a request to the model. ADDED next to "Interactive mode"; the in-flight change `add-agent-loop` also edits that requirement, but this delta does not restate it, so the two compose in either archive order.

## Impact

- **Code (`nar` repo):** new package `src/nar/mcp/` (`naming.py`, `expand.py`, `runtime.py`, `connection.py`, `manager.py`, `tool.py`, `results.py`); `src/nar/config/models.py` (`McpServerConfig` replaces the untyped dict) and `src/nar/config/loader.py` (masking in `describe`); new `src/nar/cli/slash.py` and `src/nar/ui/mcp_view.py`; `src/nar/cli/runner.py`, `src/nar/cli/repl.py` and `src/nar/cli/main.py` (connect, register, close); `pyproject.toml` (dependency `mcp>=2.3,<3`); README; tests under `tests/` including a real stdio fixture server written with the SDK.
- **Dependencies:** `mcp>=2.3,<3` (requires Python 3.10 or newer, `nar` requires 3.11). Version 2 of the SDK depends on `httpx2` instead of `httpx`; `nar`'s own `httpx` use (web tools) is unaffected and both can be installed together. No other new dependency; the demo's fallback needs Node.js only if the user chooses it.
- **Configuration:** new keys under `[mcp_servers.<name>]` (full list and defaults in `design.md`); no other setting changes.
- **Interfaces other changes rely on:** RRG-78 replaces `nar.cli.slash` with its command router and keeps `/mcp` as one registered command; RRG-72 hooks and RRG-73 subagents see MCP tools as ordinary registry tools.
- **User-visible:** `nar` start-up connects configured servers (a one-line note while connecting, one warning per failure); the model sees `mcp__linear__list_issues` and similar tools; the first call of each asks for approval at `ask`; a `/` line in interactive mode no longer goes to the model.
