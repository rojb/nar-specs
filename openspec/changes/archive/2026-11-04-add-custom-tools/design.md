# Design

## Context

`create_agent` (`src/nar/cli/runner.py`) builds the registry with `create_default_registry(cwd)`, then MCP tools are added by `connect_mcp`. `ToolRegistry.register` rejects bad names, duplicates and bad schemas with `ToolError`, and `execute` already turns a tool's exceptions into error results. `Tool.requires_approval` defaults to `False`. The approval gate covers any tool, using `Tool.preview`. `TerminalPrompter` shows how `nar` decides whether it can ask (both streams are terminals) and reads a typed answer.

Constraints:
- Importing a Python file runs it, so the decision must come before any import.
- `.nar/config.toml` comes with the repository, so nothing in it may decide trust. This includes `approval`.
- Under `auto-edit`, `.nar` is always asked about (RRG-69), so the agent cannot add a tool file without the user seeing it.

## Goals / Non-Goals

**Goals:**
- A repository cannot run code just because `nar` was opened in it.
- A trusted project does not ask again until its tool files change.
- No import cost when there are no tool files.

**Non-Goals:**
- A sandbox. Once allowed, a tool is ordinary Python with the user's permissions.
- Protection from a module a tool file imports from elsewhere: the fingerprint covers `.py` files directly in the folder. `nar` does not add the folder to `sys.path`, so sibling imports fail unless the file does it itself, in which case the user has already allowed that file to run.

## Decisions

### 1. Layout
```
src/nar/tools/custom.py     NEW       find_tool_files, fingerprint, TrustStore, load_tool_file, load_custom_tools
src/nar/ui/trust_prompt.py  NEW       TrustPrompter
src/nar/cli/runner.py       MODIFIED  create_agent loads custom tools
src/nar/cli/main.py         MODIFIED  --trust-project-tools, passed to run_single and repl.run
src/nar/cli/repl.py         MODIFIED  passes it to create_agent
examples/tools/word_count.py NEW      the example
```
No new configuration keys, on purpose (see Context).

### 2. Loading
`importlib.util.spec_from_file_location` with a module name made from a hash of the path, so two `word_count.py` files do not collide, and `exec_module`. `sys.dont_write_bytecode` is set around it so no `__pycache__` appears in a repository. The module is not left in `sys.modules`.

Classes found: `inspect.getmembers(module, inspect.isclass)`, subclasses of `Tool`, `__module__` equal to this module's name (so imports do not count), not abstract. Each is created without arguments.

Everything except `KeyboardInterrupt` is caught around the import and creation, including `SystemExit`, which is a `BaseException`. **Why not only `Exception`:** a stray `sys.exit` in a tool file must not close `nar`. `ToolError` from `register` becomes the warning for a repeated name, a bad name or a bad schema. A missing `name`, `description` or `parameters`, or a `description` that is not text, is rejected at load time: the registry would accept some of these, and `definitions()` would then fail on every model call.

### 3. Trust
`~/.nar/trusted_tools.json`: `{"version": 1, "projects": {"<resolved folder>": "<sha256>"}}`. The fingerprint is a SHA-256 over each file's name and bytes, in name order, for the `.py` files that would be loaded. Written atomically with mode 0600, with `write_private`.

**Why contents and not only the folder:** `git pull` or an edit could otherwise add code that runs with no question. **Alternative rejected:** a trust flag in `.nar/config.toml`, which the repository itself could set. **Alternative rejected:** making `yolo` imply trust, since `approval` can also come from that file.

An invalid file counts as nothing remembered, with a warning; it is overwritten only when the user allows a project, so a mistake does not silently stay.

### 4. The question
`TrustPrompter(console)`, same idea as `TerminalPrompter`: `available()` checks both streams, `ask(folder, files)` prints the list and `Run these files as code with your permissions? [y]es / [n]o: `, asking again on an unknown answer. An empty answer never allows.

Sequence in `load_custom_tools(registry, cwd, nar_home, prompter, trust_flag, note, warn)`:
1. Find global files; load them.
2. Find project files, unless the folder is the same as the global one. None: stop.
3. Fingerprint. Matching remembered answer or flag: allowed. Else `prompter.available()` false: warn and stop. Else ask.
4. Allowed: remember, then load. Refused: a line saying so, and stop.

`create_agent` calls this before the provider is used and after the provider is created, so a missing API key still fails first and no file runs in that case.

### 5. Approval
Left to the tool's own `requires_approval`, default `False`, like built-in tools. **Alternative considered:** forcing approval on every custom tool, as for MCP tools. Rejected for allowed tools: MCP servers are remote and their hints are not trusted, while a custom tool is code the user already allowed to run with full permissions, so asking on each call adds no protection against the file itself. The README tells authors to set it for tools that change things.

### 6. Display
`custom tool loaded: word_count (.nar/tools/word_count.py)` as a note on standard error, once per tool. It is printed only when custom tools exist, so existing output does not change.

## Risks / Trade-offs

- **A user says yes without reading** → the question names each file and says what it means. There is no way to verify intent beyond that.
- **Code outside the fingerprint** → a file can import from elsewhere after being allowed. Documented in Non-Goals; a changed file asks again.
- **Fingerprint race** between hashing and importing → the window is small, and whoever can write there can already change the files later. Accepted.
- **A global tool runs in every project** → the user put it there, and the README says so.

## Migration Plan

Additive. No tool files, no change in behavior.

## Open Questions

None.
