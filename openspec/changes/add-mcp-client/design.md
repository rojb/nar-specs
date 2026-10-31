# Design

## Context

After RRG-62 to RRG-69 the `nar` package has: `src/nar/tools/` (`Tool` with `requires_approval`, `preview()`, `is_project_edit()`; `ToolRegistry`, which validates arguments against each tool's JSON Schema, runs the tool, truncates output to 30,000 characters and accepts names matching `^[a-zA-Z][a-zA-Z0-9_]{0,63}$`), `src/nar/agent/` (`Agent`, a **synchronous** generator loop; `ApprovalGate`), `src/nar/config/` (`NarConfig` with `mcp_servers: dict[str, dict[str, Any]]`, "loosely typed until RRG-74"), `src/nar/ui/` (`AgentView`, `ToolRenderer`) and `src/nar/cli/` (`main.py`, `repl.py`, `runner.py`). Motivation and scope: see `proposal.md`. Required behavior: see `specs/mcp-client` and the deltas for `configuration` and `cli-entrypoint`.

Facts that shape the design (checked in the code and in the SDK):
- The agent loop, the tools and the registry are synchronous (`add-llm-providers` design: "RRG-74 can run its own event loop in a thread"). The `mcp` SDK is asyncio/anyio only.
- `create_agent` (runner) builds provider, registry and gate; `repl.run` and `run_single` call it. Nothing today needs a "close at exit" step.
- `Tool` already has the two hooks MCP needs: `requires_approval = True` and an overridable `preview`. `ApprovalGate` has no list of tool names, so MCP tools are covered with no change in `nar.agent`.
- `ToolRegistry.register` raises `ToolError` for a duplicate or invalid name and checks the schema with `Draft202012Validator`; the registry has no `unregister`.
- `mcp` on PyPI is at **2.3.0** (2026-10-02, `requires-python >=3.10`). Version 2 renamed result fields to snake_case (`is_error`, `structured_content`, `next_cursor`, `input_schema`), replaced `streamablehttp_client` with `streamable_http_client` taking an `httpx2.AsyncClient`, removed `httpx` from its dependencies, and added a high-level `Client` that accepts `StdioServerParameters` or a transport. The stdio transport on Windows resolves `.cmd`/`.bat`/`.exe` extensions of the command itself (`mcp/os/win32/utilities.py` tries `shutil.which(command)` and then the common script extensions), so `npx` works without a shell. The stdio child receives only an allow-list of environment variables (on Windows `APPDATA`, `PATH`, `PATHEXT`, `SYSTEMROOT`, `TEMP`, `USERPROFILE`, ...) plus the `env` given.
- `CallToolResult.content` is a list of `TextContent | ImageContent | AudioContent | ResourceLink | EmbeddedResource`; `structured_content` is optional; `is_error` marks a tool-level error. A server-side protocol error raises `MCPError` on the client.
- `Client(...)` is an async context manager; anyio cancel scopes must be entered and exited **in the same task**.

Sources: SDK docs on transports and headers https://py.sdk.modelcontextprotocol.io/client/transports/ ; v1 to v2 migration guide https://py.sdk.modelcontextprotocol.io/migration/ ; package page https://pypi.org/project/mcp/ ; Linear's MCP documentation https://linear.app/docs/mcp .

## Goals / Non-Goals

**Goals:**
- Any MCP tool is usable by the model as an ordinary tool and is covered by approval without special cases in `nar.agent`.
- A broken, slow or hostile server cannot hang, crash or poison `nar`: bounded waits everywhere, errors become messages.
- Secrets stay out of config files, terminal output and the model's context.
- The sync/async boundary lives in one small module, so no other module knows asyncio exists.
- A student on Windows can reproduce the demo with a Linear API key and no extra software.

**Non-Goals:**
- Resources, prompts, sampling, elicitation, roots, progress display, `tools/list_changed` refresh, reconnection (Decision 4), OAuth inside `nar` (Decision 6), per-tool approval policy, a `.env` loader, and the full slash-command router (Decision 9).

## Decisions

### 1. Module layout and configuration keys
```
src/nar/
├── mcp/                      NEW package (imported only when an enabled server exists)
│   ├── __init__.py           NEW   re-exports McpManager, ServerStatus
│   ├── naming.py             NEW   sanitize(), tool_name(server, tool, taken)
│   ├── expand.py             NEW   expand_env(value, environ) -> str; raises MissingVariable(name)
│   ├── runtime.py            NEW   McpRuntime: event loop thread, submit(), run(), stop()
│   ├── connection.py         NEW   ServerConnection: one task per server holding Client; list_tools(), call_tool()
│   ├── results.py            NEW   to_tool_result(CallToolResult) -> ToolResult
│   ├── tool.py               NEW   McpTool(Tool)
│   └── manager.py            NEW   McpManager: connect_all(), register_tools(), statuses(), close()
├── config/
│   ├── models.py             MODIFIED  McpServerConfig; NarConfig.mcp_servers: dict[str, McpServerConfig]
│   ├── loader.py             MODIFIED  describe() masks env/headers literals
│   └── paths.py              MODIFIED  mcp_log_dir() -> nar_home()/"mcp-logs"
├── cli/
│   ├── slash.py              NEW   command table, parse_slash(), dispatch(), register_command()
│   ├── repl.py               MODIFIED  connects servers, dispatches slash lines, closes on exit
│   ├── runner.py             MODIFIED  connect_mcp() context manager used by repl and run_single
│   └── main.py               MODIFIED  run_single wraps the request in connect_mcp()
└── ui/
    ├── mcp_view.py           NEW   render_mcp_status(statuses, console)
    └── agent_view.py         MODIFIED  note(message): dim line on standard error
pyproject.toml                MODIFIED  dependency "mcp>=2.3,<3"
README.md                     MODIFIED  "MCP servers" section and the Linear demo
tests/test_mcp_naming.py, test_mcp_expand.py, test_mcp_results.py, test_mcp_runtime.py,
tests/test_mcp_connection.py, test_mcp_manager.py, test_mcp_tool.py, test_slash.py, test_mcp_view.py   NEW
tests/fixtures/mcp_demo_server.py   NEW  stdio server written with the SDK (tools: echo, slow, big, image, fail, broken_schema, many)
tests/test_config.py, test_cli.py, test_agent_view.py, test_agent_loop.py   MODIFIED
```
`nar/agent/*`, `nar/providers/*`, `nar/context/*`, the other tools and `nar/tools/registry.py` are **not** touched: the approval gate and the registry already do everything needed.

**New configuration keys** (`~/.nar/config.toml` and `.nar/config.toml`), all under `[mcp_servers.<name>]`:

| Key | Type | Default | Meaning |
|---|---|---|---|
| `command` | string | none | stdio: program to start (`${VAR}` expanded) |
| `args` | list of strings | `[]` | stdio: arguments (`${VAR}` expanded) |
| `env` | table of strings | `{}` | stdio: extra environment of the child (`${VAR}` expanded) |
| `url` | string | none | HTTP: streamable HTTP endpoint (`${VAR}` expanded) |
| `headers` | table of strings | `{}` | HTTP: headers sent with every request (`${VAR}` expanded) |
| `enabled` | bool | `true` | `false` skips the server entirely |
| `enabled_tools` | list of strings | all tools | allow-list of the server's own tool names |
| `startup_timeout` | number (s, > 0) | `30` | connecting plus listing tools |
| `tool_timeout` | number (s, > 0) | `60` | one tool call |

Exactly one of `command` / `url`; `args`/`env` with `url` and `headers` with `command` are rejected (Pydantic `extra="forbid"` plus a model validator whose message names the server and key). No new top-level key. Server names: `^[A-Za-z0-9][A-Za-z0-9_-]{0,31}$`, and two names equal after `-` to `_` are rejected (they would produce the same registered prefix). Constants: `MAX_DESCRIPTION_CHARS = 2000`, `MAX_TOOL_NAME = 64`, `CLOSE_GRACE_SECONDS = 5`.

### 2. SDK version: `mcp>=2.3,<3` and the high-level `Client`
Use the current major version. Version 1 is still published (1.30) and many online examples use it (`ClientSession`, `streamablehttp_client`, camelCase fields), but starting a new integration on a line the maintainers have superseded means a forced migration within the term, and the tutorial APIs differ enough that copied snippets would fail. Version 2's `Client` hides session construction, initialization and the transport pairing, which is exactly the surface a thin wrapper wants. Pin the upper bound because 2.x is young and the migration guide lists many renames. Consequences: `httpx2` (not `httpx`) is the HTTP library for the MCP transport, so `connection.py` builds `httpx2.AsyncClient(headers=..., timeout=httpx2.Timeout(30.0, read=...))` itself; `nar`'s web tools keep using `httpx`; both packages install side by side. `mcp` requires Python 3.10+, `nar` requires 3.11.
**Alternative rejected:** `mcp>=1.x` pinned below 2: more tutorials, but it is the superseded line. **Alternative rejected:** writing a JSON-RPC client by hand: the course asks for the official SDK.
All SDK imports are confined to `connection.py` (and `httpx2`), imported lazily, so `nar` without MCP servers does not pay the import cost and unit tests of naming, expansion and result mapping run with fake objects.

### 3. Sync/async boundary: one event-loop thread, one task per server (`runtime.py`)
`McpRuntime` starts a **daemon thread** that creates an event loop (`asyncio.new_event_loop()`; the Proactor loop on Windows supports subprocesses) and runs it forever. API:
- `submit(coro) -> concurrent.futures.Future` (`asyncio.run_coroutine_threadsafe`).
- `run(coro, timeout)`: submit, then wait **by polling** `future.result(timeout=0.2)` in a loop until done or `timeout` elapsed. Reason: on Windows a plain `Future.result()` blocks in a lock wait that Ctrl+C does not interrupt; a short poll lets `KeyboardInterrupt` surface within 200 ms. On `KeyboardInterrupt` or timeout the future is cancelled (`future.cancel()` cancels the running task in the loop) and the exception propagates / is converted by the caller.
- `stop(grace)`: ask the loop to cancel its tasks, `loop.call_soon_threadsafe(loop.stop)`, join the thread for at most `grace` seconds.
`ServerConnection` runs its whole life in **one task** (anyio requires enter and exit of `Client` and of the transport context in the same task):
```
async def _serve(self):                       # task per server
    try:
        async with AsyncExitStack() as stack:
            transport = stdio_client(params, errlog=log_file)  |  streamable_http_client(url, http_client=<httpx2 client>)
            client = await stack.enter_async_context(Client(transport, read_timeout_seconds=tool_timeout))
            tools = await _list_all(client)          # follows next_cursor
            self._ready.set_result(tools)
            await self._stop.wait()                  # served until close()
    except BaseException as error:
        if not self._ready.done(): self._ready.set_exception(error)
        else: self._mark_failed(error)               # server died later
```
`call_tool(name, args)` is a separate coroutine submitted to the loop (`await client.call_tool(...)`); it runs as its own task, so cancelling it does not touch the serving task and the connection survives (spec "Cancelling a call"). The agent thread calls `McpTool.execute -> manager.call(...) -> runtime.run(...)` and so stays synchronous. No other module imports asyncio.
**Alternatives rejected:** (a) `asyncio.run()` per call, re-connecting each time: a stdio server would be started for every call (seconds, loses server state), and an HTTP server would re-authenticate. (b) Making the agent loop async: touches the providers, the loop, tests and every caller for no benefit to the user (the argument of `add-llm-providers` still holds). (c) `anyio.from_thread` portals: equivalent to the thread, with a heavier API and Windows signal quirks. (d) Running the loop on the main thread and the agent in a worker: Ctrl+C would then be delivered to the wrong thread.

### 4. Connecting, failures and shutdown (`manager.py`, `connection.py`)
`McpManager.connect(configs, environ, view_note, warn)`:
1. Select enabled servers; if none, return an empty manager **without importing the SDK or starting the thread**.
2. `note("connecting MCP servers: a, b ...")`.
3. For each server, expand `${VAR}` (Decision 5). A missing variable marks that server `failed` immediately, with no process or request.
4. Start the runtime and submit one `ServerConnection` per server; wait for all `ready` futures together (`asyncio.gather(..., return_exceptions=True)` inside one coroutine run through `runtime.run`), each wrapped in `asyncio.wait_for(ready, startup_timeout)`. On timeout the task is cancelled, which closes the transport and ends the child process.
5. Per server: success -> `connected` with its tool list; exception -> `failed` with a **reason string** from `describe_failure(error)` and one `warn` line. The reasons are short and never include header or environment values: `FileNotFoundError` -> `command 'x' not found`; `TimeoutError` -> `timed out after N s`; an `ExceptionGroup` (anyio task groups) is reduced to its first leaf; HTTP status 401/403 detected from the SDK error text -> `authentication rejected (HTTP 401)`; any other exception -> `<ExceptionType>: <message>` with the message cut at 200 characters and any substring equal to an expanded secret replaced by `***` (the manager knows the expanded values and scrubs them as a last line of defense).
6. A stdio child's standard error goes to `~/.nar/mcp-logs/<server>.log` (opened in text mode, truncated on each start; `NAR_HOME` respected). This keeps `npx` download chatter and server logs out of the terminal and the answer stream and gives `/mcp` something to point at.
7. `KeyboardInterrupt` during step 4 cancels the wait, calls `close()` and re-raises; the CLI error boundary prints `cancelled` and exits 130.

**Late failure:** when a connection task ends after `ready` (child exited, HTTP connection dropped), `_mark_failed` records the error; `ServerConnection.call_tool` then returns `ServerGone`, which `McpTool` turns into `ToolResult.error("MCP server 'x' is no longer connected: <reason>. Restart nar to reconnect.")`. **No automatic reconnect** (non-goal): a reconnect policy (backoff, re-listing tools, re-registering names) is its own design, and a restart takes seconds. `/mcp` reads the live state.

**Shutdown:** `McpManager.close()` (idempotent) sets every connection's stop event so each task leaves its `async with` normally (the SDK closes the child's stdin and, if it does not exit, terminates it, including the process tree on Windows), waits up to `CLOSE_GRACE_SECONDS` for all tasks, then stops the loop thread. The thread is a daemon, so a stuck server can never keep the interpreter alive; the worst case is a process the SDK could not terminate, which the spec accepts only for the pathological case and the manual check verifies for normal ones. `connect_mcp()` in `runner.py` is a context manager that calls `close()` in `finally`, so exit, end of input, Ctrl+C and exceptions all stop the servers.

### 5. Environment references (`expand.py`)
`expand_env(text, environ)` replaces `${NAME}` (regex `\$\{([A-Za-z_][A-Za-z0-9_]*)\}`) with `environ[NAME]`; unset or empty raises `MissingVariable(NAME)`. Applied to `command`, each `args` item, `env` values, `url` and `headers` values, **at connect time**, never when loading the configuration: `--show-config` and the config model keep the placeholders, so nothing secret is held in `NarConfig` and a config object can be logged safely. No default syntax (`${VAR:-x}`) and no escape for a literal `${`: YAGNI, and a literal `${` in a header is not a realistic need; revisit if it appears. `env` values for a stdio server therefore support `LINEAR_API_KEY = "${LINEAR_API_KEY}"` to forward one variable explicitly while the rest of the user's environment is withheld by the SDK.
**Masking** in `LoadedConfig.describe`: for flattened keys matching `^mcp_servers\.[^.]+\.(env|headers)\.`, a value without `${` is printed as `********`. Tested for `--show-config` output and for the `/mcp` view (which never prints header or env values at all).
**Alternatives rejected:** a separate `env_headers = { Authorization = "LINEAR_API_KEY" }` key (as some tools do): it cannot express `Bearer ${KEY}` and adds a second concept; reading the token from `NAR_*` environment variables implicitly: hidden magic; a `.env` file loader: out of scope and an easy way to commit secrets.

### 6. Connecting to Linear (research result and choice)
**Facts (Linear's documentation, https://linear.app/docs/mcp):** the official remote server is `https://mcp.linear.app/mcp` using **Streamable HTTP** as its primary transport (`/mcp/readonly` exposes only read tools; `/sse` is a deprecated fallback). Interactive setup uses OAuth 2.1 with dynamic client registration, but the server "accepts tokens directly in the `Authorization: Bearer <yourtoken>` header" instead of the interactive flow; a Linear personal API key (created in Linear under Settings > Account > Security & access, optionally restricted to the `Read` permission) is one of those tokens. For clients without remote support Linear documents `npx -y mcp-remote https://mcp.linear.app/mcp`, which performs the OAuth login in a browser. A search of third-party guides shows the same header configuration (`"Authorization": "Bearer ${LINEAR_API_KEY}"`) used with an API key. This was not tested end to end while writing the design (it needs the owner's key), so the manual verification task checks it first and the fallback below is ready.

**Choice (primary): streamable HTTP with `Authorization: Bearer ${LINEAR_API_KEY}`.**
```toml
[mcp_servers.linear]
url = "https://mcp.linear.app/mcp"
headers = { Authorization = "Bearer ${LINEAR_API_KEY}" }
```
The user provides: a Linear **personal API key** in the environment variable `LINEAR_API_KEY` (`setx LINEAR_API_KEY "lin_api_..."` then a new terminal on Windows, or `$env:LINEAR_API_KEY=...` for one PowerShell session, `export` in Git Bash). Nothing else: no Node.js, no browser, works over SSH and in CI, one config block, deterministic tests with a local server. A read-only key (or the `/mcp/readonly` URL) is the safest way to run the demo; the "done when" (list issues) only needs read access. Because every MCP call is approval-gated anyway, write tools are never run unasked at the default level.

**Fallback (documented, not default): OAuth through `mcp-remote` over stdio.**
```toml
[mcp_servers.linear]
command = "npx"
args = ["-y", "mcp-remote", "https://mcp.linear.app/mcp"]
startup_timeout = 120
```
The user provides: Node.js 18+ (for `npx`) and a browser for the one-time OAuth login on first start (tokens are cached by `mcp-remote`); `startup_timeout` is raised because the first start waits for the login. It exercises the stdio transport against a real service and covers the case where Linear does not accept the user's key on the remote endpoint. Windows quirk: `npx` is `npx.cmd`; the SDK resolves it (Context), and the manual verification confirms it on the real machine.

**Alternatives rejected:** implementing OAuth 2.1 with dynamic client registration inside `nar` (the SDK has client-side OAuth helpers, but a local callback server, token storage and refresh are a feature of their own, for a demo an API key meets); a community stdio Linear server driven by `LINEAR_API_KEY` (not evaluated; a third-party package with write access to the team's workspace is a supply-chain risk the official server avoids); Atlassian/Jira (needs an Atlassian cloud site and OAuth) and Microsoft Teams (needs an Azure tenant): the team already uses Linear, so its project is real data.

### 7. Tool names (`naming.py`)
`tool_name(server, tool, taken) -> str`:
1. `prefix = "mcp__" + sanitize(server) + "__"`, `sanitize(s) = re.sub(r"[^A-Za-z0-9_]", "_", s)`.
2. `name = prefix + sanitize(tool)`. If `len(name) <= 64` and `name not in taken`, use it.
3. Otherwise `suffix = "_" + sha1(f"{server}\0{tool}")[:6]`; `name = (prefix + sanitize(tool))[: 64 - len(suffix)] + suffix`. This covers both overlong names and collisions after sanitizing, deterministically (same input, same name across runs, so approvals and the model's habits are stable).
4. A name still in `taken` after step 3 (a 24-bit hash collision) is skipped with a warning; the probability is negligible and a loop would hide a real bug.
Server names are limited to 32 characters (Decision 1) so the prefix is at most 40 characters and at least 24 remain for the tool part. The pattern the registry enforces (`^[a-zA-Z][a-zA-Z0-9_]{0,63}$`) rules out `-` and `.`, which several providers also reject; no registry change is needed. The original tool name is kept on `McpTool` for the server call and for the preview.
**Alternative rejected:** raising the registry pattern to allow `-`: some providers' rules (Anthropic allows `-`; OpenAI-compatible endpoints vary) differ, and the registry pattern is the shared safe subset.

### 8. `McpTool`, results and approval (`tool.py`, `results.py`)
- `McpTool(Tool)`: `name` = registered name, `description` = `f"[MCP server {server}] {text}"` where `text` is the server description (or title, or the tool name) cut to `MAX_DESCRIPTION_CHARS` with a `... [description shortened]` marker, `parameters` = `input_schema` deep-copied with a top-level `$schema` key removed (some providers reject it; the registry validates with draft 2020-12 anyway), `requires_approval = True`.
- **Schema passthrough and validation:** the registry's `register` already calls `_check_schema` (type object, valid Draft 2020-12); `McpManager.register_tools` catches `ToolError`, warns `warning: MCP tool 'x' of server 's' skipped: <reason>` and moves on. Valid arguments are checked by the registry's `validate_arguments` before `execute`, so the server only receives calls that match its own schema (spec scenario). Schemas that use other drafts' keywords are accepted when the validator accepts them; this is passthrough, not normalization (Risks).
- `execute(arguments, context)`: `manager.call(server, original_name, arguments, timeout=tool_timeout)` through `runtime.run`. Outcomes: result -> `to_tool_result`; `MCPError` -> `ToolResult.error(f"MCP error {code}: {message}")`; timeout -> `ToolResult.error("MCP tool 'x' timed out after N s")`; `ServerGone` -> the Decision 4 message; `KeyboardInterrupt` propagates unchanged (cancels the call task first, Decision 3). Any other exception is already converted to an error result by `ToolRegistry.execute`.
- **Result mapping (`to_tool_result`):** text blocks joined with `\n\n`; `ImageContent`/`AudioContent` -> `[image image/png, N bytes, not shown]` (N from the decoded base64 length, computed arithmetically, not by decoding); `ResourceLink` -> `[resource link: <name> <uri>]`; `EmbeddedResource` with text -> that text, with a blob -> `[binary resource <uri> <mime>, N bytes, not shown]`; no blocks but `structured_content` -> `json.dumps(..., ensure_ascii=False)`; nothing -> `(no output)`; `is_error` -> `ToolResult.error(text)`. `summary` = `f"{server}: {tool}"` for success. Images are described, not forwarded, because `ToolResult.output` is text and the provider adapters have no image path; forwarding images is an extension for when a provider path exists. Output is cut by `ToolRegistry._finish` (30,000 characters), the same rule as every tool.
- **Approval:** `preview()` is overridden only to name the origin: `ToolPreview(title=f"MCP {server}: {original_name}", detail=json.dumps(arguments, indent=2, ensure_ascii=False, default=str))`. It never calls the server. `requires_approval = True` and the default `is_project_edit = False` mean: asked at `ask` and `auto-edit`, run at `yolo`, session approval keyed by the registered name. Server annotations (`readOnlyHint`, ...) are ignored on purpose: they are self-declared by software the user may not trust (a malicious or compromised server can claim read-only on a destructive tool), and the approval system exists for exactly that doubt. The cost is a question per distinct tool at `ask`; `a` ("always this tool this session") makes it one question per tool, and `enabled_tools` keeps the tool list short. A trust list in the configuration is the follow-up if this proves annoying.
- **Timeouts:** `tool_timeout` is enforced on the loop side with `asyncio.wait_for(client.call_tool(...), tool_timeout)` and again as the `runtime.run` wait limit (`tool_timeout + 5` s) as a backstop.

### 9. `/mcp` and the slash-command seam (`cli/slash.py`, `ui/mcp_view.py`)
**Decision: implement `/mcp` now, behind a deliberately tiny command table, and do not add a second CLI surface (`nar mcp list` / `--mcp`).** RRG-78 `add-tui-commands` (Backlog) will own the router, completion and `/help /clear /exit /model /approval /config /tools /tokens`. Waiting for it would leave RRG-74's issue text ("`/mcp` command") unmet and the course reviewer's expectation of `/mcp` unshown; building the router here would duplicate RRG-78 and decide its design (prompt_toolkit) for it. So:
```python
# nar/cli/slash.py
Handler = Callable[[SlashContext, str], None]       # (context, argument text)
def register_command(name: str, handler: Handler, summary: str) -> None
def parse_slash(line: str) -> tuple[str, str] | None   # regex ^/([A-Za-z][\w-]*)(?:\s+(.*))?$
def dispatch(line: str, context: SlashContext) -> bool # True when the line was a command
@dataclass SlashContext: mcp: McpManager | None; console: Console
```
`repl.run` calls `dispatch` before sending a line to the agent; `dispatch` handles unknown names (error plus the list of registered commands). RRG-78 replaces the body of `parse_slash`/`dispatch` with its router and keeps `register_command(name, handler, summary)` as its registration entry point (its proposal says "registro abierto"); `/mcp` is registered by `nar.cli.slash` at import with `render_mcp_status`. A path such as `/etc/hosts` does not match the regex (a second `/` right after the name), so it still reaches the model.
**Why no `nar mcp list` / `--mcp`:** `nar` takes a positional `prompt`; a Click sub-command group would make `nar mcp` ambiguous with a prompt (the same reason `--show-config` is an option). `/mcp` inside a session shows live state, which is what a user debugging a connection needs; `nar --show-config` already shows the static configuration without starting servers. If a non-interactive status is wanted later it is a one-line option calling the same view.
**View:** `render_mcp_status(statuses, console, only=None)`. Per server one header line `name  (stdio|http)  connected|failed|disabled`, then for connected servers `N tools` and the registered names (sorted, one per line, dim), for failed ones the reason and `log: <path>` for stdio. All text is printed literally (`markup=False` / `Text`) because tool names and reasons come from external software. The output goes to standard output (it is the command's answer). `ServerStatus(name, transport, state, tools: tuple[str,...], error: str | None, log_path: Path | None)` is a frozen dataclass produced by `manager.statuses()` from live state.

### 10. Wiring (`cli/runner.py`, `cli/repl.py`, `cli/main.py`)
```python
@contextmanager
def connect_mcp(config, agent, view) -> Iterator[McpManager | None]:
    manager = McpManager.connect(config.mcp_servers, os.environ, note=view.note, warn=view.warn)
    try:
        manager.register_tools(agent.registry, warn=view.warn)
        yield manager
    finally:
        manager.close()
```
`create_agent` is unchanged (the provider is created first, so a missing API key fails before any server is started). `run_single` becomes `agent = create_agent(...)`; `with connect_mcp(...)`: `run_request(...)`. `repl.run` wraps the whole read loop in the same `with` and passes `manager` into `SlashContext`. With no enabled server `connect_mcp` yields a manager whose `register_tools`/`close` do nothing and which never imported the SDK. Ctrl+C at the prompt or during a request ends the `with` through the normal path, and `main`'s error boundary keeps its exit codes. Startup order for the user: `Using AGENTS.md ...`, then `connecting MCP servers: linear ...`, warnings if any, then the banner line.

### 11. Tests
All offline except one opt-in live test. A real stdio fixture server (`tests/fixtures/mcp_demo_server.py`, SDK `MCPServer` with tools `echo`, `slow`, `big`, `image`, `fail`, `structured`, `many` and a tool with a broken schema when started with a flag) is started through the real `ServerConnection`, because mocking the SDK would test nothing about the Windows process handling. A streamable HTTP test serves the same server with `run("streamable-http")` on a free localhost port in a subprocess and records request headers; a tiny `http.server` that answers 401 covers authentication failures. Slow-server and hanging-close behavior use fixture flags. Unit tests use fakes for the manager (`/mcp` view, slash dispatch, `McpTool` with a fake connection). `test_mcp_tool.py` adds the approval path through `ApprovalGate` with a `ScriptedPrompter` (asked at `ask` and `auto-edit`, not at `yolo`, rejected call never reaches the connection). The live Linear test is marked `@pytest.mark.skipif(not os.environ.get("LINEAR_API_KEY"))` and is only run by the owner. Expected test counts per task are in `tasks.md`.

### 12. README
Section "MCP servers": configuration keys table, secrets with `${VAR}`, `/mcp`, approval behavior of MCP tools, `enabled_tools`, troubleshooting (log path, `startup_timeout`, `npx` on Windows), and "Demo: Linear" with the primary and fallback configurations of Decision 6, exact commands for Git Bash and PowerShell to set the key, the read-only URL tip, and a sample session that ends with the issue list. The reserved-for-RRG-74 note in the settings table is replaced.

## Risks / Trade-offs

- [Linear does not accept the user's API key at the remote endpoint, or requires a different header form] -> Documented fallback via `mcp-remote` (stdio, OAuth), checked in the manual verification; the code path is the same for both transports, so only configuration changes. The 401 failure message tells the user the authentication was rejected, not "something failed".
- [A malicious or buggy server returns huge output, or a tool description that contains instructions (prompt injection)] -> Output cut at 30,000 characters, descriptions at 2,000; every call is shown and approved by the user at the default level; read-only hints are not trusted; `enabled_tools` limits exposure. Prompt injection through tool results is inherent to MCP and is not solved here; the approval prompt is the control.
- [Servers with many tools (Linear lists dozens) inflate every request's token count] -> `enabled_tools` and the `/mcp/readonly` URL reduce it; descriptions are capped. No automatic pruning; context management is another change.
- [Windows: a child process survives `nar`] -> The SDK terminates the process tree on Windows; the manual verification checks for leftover `node`/`python` processes after exit and after Ctrl+C; `mcp-remote` through `npx.cmd` is the case most likely to leave a grandchild, so it is checked explicitly.
- [Ctrl+C is not delivered promptly while the main thread waits] -> Polling waits of 200 ms (Decision 3), covered by a test that interrupts a 60 s call via a timer-raised `KeyboardInterrupt`.
- [`mcp` 2.x is new: API drift between 2.3 and later 2.x] -> Upper bound `<3`; only a handful of SDK symbols are used and all are confined to `connection.py`; the stdio fixture test fails loudly on drift.
- [Schemas with newer or older JSON Schema dialects fail the registry's draft 2020-12 check] -> The tool is skipped with a warning naming it, not the whole server. If common servers hit this, relaxing the check for MCP tools is a one-function change in `register_tools`.
- [Two `httpx` generations installed] -> Separate package names (`httpx` and `httpx2`); no shared global state. The SDK's TLS uses the OS trust store; corporate proxies are the user's `SSL_CERT_FILE` concern and are documented in the README troubleshooting.
- [Startup latency: each enabled server adds up to `startup_timeout` seconds before the prompt] -> Servers connect concurrently; a one-line note shows what `nar` is waiting for; `enabled = false` and Ctrl+C are available. Lazy connecting on first use was rejected: the model needs the tool list in its first request, and failures should surface at start, not mid-conversation.
- [A `/`-prefixed line used to go to the model and now may be an unknown command] -> Only lines matching the command form are affected, the error lists commands, and paths still pass through.

## Migration Plan

No data migration. Existing configurations with an empty or absent `mcp_servers` behave exactly as before and do not import the SDK. A configuration that already contained untyped `mcp_servers` entries (reserved earlier, never read) is now validated, so entries must follow Decision 1; none are known to exist. `pip install -e ".[dev]"` installs `mcp`. Rollback: revert the PRs; `mcp_servers` returns to the untyped reserved table.

## Open Questions

- Does Linear accept a personal API key as a Bearer token on `https://mcp.linear.app/mcp` for the team's workspace (`rgonaut`)? Documented as supported; confirmed or refuted by the first manual verification step, with the fallback ready.
- Should a trust list (`approve_tools = ["mcp__linear__list_issues"]`) exist so read-only MCP tools stop asking? Deferred with allow-lists in general; no spec change needed beyond a new requirement.
- Should `tools/list_changed` and reconnect be supported? Only if a server the team uses needs it; today a restart is enough.
