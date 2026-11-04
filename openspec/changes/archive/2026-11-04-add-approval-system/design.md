# Design

## Context

After RRG-62 to RRG-68 the `nar` package has: `src/nar/tools/` (`Tool` with `requires_approval`, `ToolContext(cwd)`, `ToolResult` with `output`, `is_error`, `summary`, `diff`, `display` in `base.py`; `ToolRegistry` in `registry.py`, which keeps the context privately and is the single place that validates and runs a call; `files.py` with `resolve_path`; the file, search, web, planning/memory and `shell` tools), `src/nar/agent/loop.py` (`Agent`, `ToolGate`, `approval_gate`), `src/nar/ui/` (`AgentView`, `ToolRenderer`) and `src/nar/cli/` (`main.py`, `repl.py`, `runner.py`). Motivation and scope: see `proposal.md`. Required behavior: see `specs/approval-system` and the deltas for `agent-loop`, `tool-framework`, the three tool specs, `configuration` and `cli-entrypoint`.

Facts that shape the design (checked in the code):
- `Agent._run_tool` calls `self.gate(tool, arguments)` **after** the `ToolCallStarted` event was yielded and handled. `AgentView.handle` has by then printed the call line and stopped the waiting spinner (`_stop_waiting()` runs for every event except `ModelCallStarted`), so a gate that prints and reads input runs with a quiet terminal and needs no change in the loop or the view's spinner logic. The next `ModelCallStarted` restarts the spinner.
- A gate that returns a `ToolResult` replaces the tool run; the loop already adds that result to the conversation like any other, so a rejection costs nothing in the loop. `KeyboardInterrupt` raised inside the gate propagates through the generator, `Agent.run`'s `finally` rolls the request back, `run_request` re-raises, and the REPL prints "cancelled" / single mode exits 130: exactly the existing cancellation path.
- `approval_gate(level)` today is a closure; the CLI builds it in `runner.create_agent` with `config.approval`. `ApprovalLevel` (`ask`, `auto-edit`, `yolo`) and the `approval` key (default `ask`) already exist in `config/models.py`. There is **no** `--approval` option in `cli/main.py` (only `--provider` and `--model`), and `load_config(cwd, overrides={...})` already merges command-line values with the source label `command line`.
- `edit_file.execute` validates, reads the file, finds the single match, builds `new_text`, writes it and then builds the diff with `_unified_diff`. Everything before the write is pure, so the preview can share it. `write_file.execute` checks encoding, directory, blocking ancestor, then creates parents and writes. `shell.execute` checks empty command, the blocklist, shell presence and cwd before spawning.
- The prompt must go to standard error (spec: standard output holds only the answer), where Rich's `Console(stderr=True)` already prints tool activity.

## Goals / Non-Goals

**Goals:**
- Approval that works for any tool, including ones registered later (MCP), without a list of tool names anywhere in the approval code.
- The loop and the tools stay free of terminal I/O: asking is a small interface (`ApprovalPrompter`) implemented once for the terminal and replaceable by a scripted fake in tests or by "nothing" for headless agents.
- Deterministic behavior when asking is impossible: never hang, never guess.
- A preview that is computed by the same code path as execution, so what the user approves is what runs.

**Non-Goals:**
- Closing the gap between preview and execution (see Risks), a sandbox, or a security boundary against a hostile model: approval is the control for honest mistakes and prompt injection that the user can notice.
- Persisting approvals, pattern allow-lists, batch approval of several calls, editing a call before approving it, or changing the level mid-session (RRG-78 will set `ApprovalGate.level`).
- Making the `.git`/`.nar` protection a general path policy; it is a single, deliberate exception to `auto-edit` (Decision 4).

## Decisions

### 1. Module layout
```
src/nar/
├── agent/
│   ├── approval.py        NEW       ApprovalDecision, ApprovalRequest, ApprovalAnswer, ApprovalPrompter (Protocol),
│   │                                ApprovalGate, approval_gate(), refusal messages
│   ├── loop.py            MODIFIED  approval_gate() and its imports removed; ToolGate type and Agent unchanged
│   └── __init__.py        MODIFIED  re-exports approval_gate, ApprovalGate, ApprovalPrompter, ApprovalDecision, ApprovalAnswer, ApprovalRequest
├── tools/
│   ├── base.py            MODIFIED  ToolPreview; Tool.preview(); Tool.is_project_edit()
│   ├── files.py           MODIFIED  is_project_path(path, cwd)
│   ├── edit_file.py       MODIFIED  _plan() shared by execute() and preview(); is_project_edit()
│   ├── write_file.py      MODIFIED  preview(); is_project_edit()
│   ├── shell.py           MODIFIED  preview()
│   └── registry.py        MODIFIED  `context` read-only property
├── ui/
│   ├── approval_prompt.py NEW       TerminalPrompter: renders a preview and reads the answer
│   └── tool_view.py       MODIFIED  ToolRenderer.show_diff() (public wrapper of the existing diff printing)
└── cli/
    ├── runner.py          MODIFIED  create_agent builds TerminalPrompter and approval_gate(level, prompter, context)
    └── main.py            MODIFIED  --approval option passed as an override
tests/test_approval_gate.py     NEW
tests/test_approval_prompt.py   NEW
tests/test_tools_base.py, test_tools_edit_file.py, test_tools_write_file.py, test_tools_shell.py,
tests/test_agent_loop.py, test_tools_registry.py, test_cli.py, test_tool_view.py   MODIFIED
README.md                       MODIFIED
```
`nar/providers/*`, `nar/context/*`, `nar/config/*` (the `ApprovalLevel` enum and the `approval` key already exist), `nar/errors.py` and `scripts/run_tool.py` are **not** touched. **New config keys: none** (`approval` in `~/.nar/config.toml` and `.nar/config.toml`, default `ask`, already exists). Constants in `agent/approval.py`: `MAX_REASON_CHARS = 500`; in `ui/approval_prompt.py`: `MAX_DETAIL_LINES = 60`, `MAX_DIFF_LINES = 100`.

Why `ApprovalGate` lives in `nar.agent` and not in `nar.ui`: the decision logic (levels, session memory, refusal texts) has no terminal in it and must be testable and reusable by subagents (RRG-73) and hooks (RRG-72) without Rich. Only reading the answer and drawing the preview is UI.

### 2. Tool API: previews and project-edit declaration (`tools/base.py`)
```python
@dataclass(frozen=True)
class ToolPreview:
    title: str                  # one line, e.g. "Edit src/app.py"
    detail: str | None = None   # command text, beginning of a new file, JSON arguments
    diff: str | None = None     # unified diff, shown with the diff colors
    error: str | None = None    # the message execute() would return; the call cannot run

class Tool(ABC):
    def preview(self, arguments: dict[str, Any], context: ToolContext) -> ToolPreview:
        # default: ToolPreview(title=self.name, detail=json.dumps(arguments, indent=2, ensure_ascii=False, default=str))
    def is_project_edit(self, arguments: dict[str, Any], context: ToolContext) -> bool:
        return False
```
- Both are plain methods with safe defaults, so a tool written by someone who never heard of approvals (an MCP wrapper) is previewed correctly as "name + arguments" and, because `is_project_edit` is `False`, asked at every level below `yolo`. **This is the extension point RRG-74 needs: set `requires_approval = True`; override `preview` only to look nicer.**
- Previews receive arguments that already passed schema validation (the gate validates first, see Decision 3) and must not write to the file system, spawn processes or use the network; reading is allowed. `preview` may raise; the gate then falls back to the default preview and shows a dim note "preview unavailable".
- `ToolPreview.error` exists so that a doomed call is not shown to the user: the gate returns `ToolResult.error(preview.error)` instead of asking. The text is the one `execute` would return because both come from the same code (Decisions 5 to 7).
- **Alternatives rejected:** (a) a `dry_run` flag on `ToolContext` that `execute` obeys: every tool would need correct dry-run branches inside code that writes, and one forgotten branch is a silent write during a "preview"; a separate method that shares only the pure planning step cannot write by construction. (b) A fixed enum on the tool (`approval_category = "edit" | "command"`): the `auto-edit` decision depends on the call (the path), so a per-call method is needed anyway. (c) Previewing in the UI from `call.arguments` only (a `match` on tool names): breaks for MCP tools and duplicates the edit logic.

### 3. The gate (`agent/approval.py`)
```python
class ApprovalDecision(StrEnum): APPROVE; APPROVE_SESSION; REJECT
@dataclass(frozen=True) class ApprovalAnswer: decision: ApprovalDecision; reason: str | None = None
@dataclass(frozen=True) class ApprovalRequest: tool_name: str; preview: ToolPreview
class ApprovalPrompter(Protocol):
    def available(self) -> bool: ...                       # can a human be asked right now?
    def ask(self, request: ApprovalRequest) -> ApprovalAnswer: ...

class ApprovalGate:
    def __init__(self, level, prompter: ApprovalPrompter | None = None, context: ToolContext | None = None)
    level: ApprovalLevel                                   # mutable on purpose (RRG-78 /approval)
    approved_tools -> frozenset[str]                       # session memory, read-only view
    def __call__(self, tool: Tool, arguments: Mapping[str, Any]) -> ToolResult | None
def approval_gate(level, prompter=None, context=None) -> ApprovalGate
```
`__call__` is the `ToolGate` of RRG-63 (same signature, so RRG-72's `before_tool` composes with it). Order of checks:
1. `not tool.requires_approval` or `level is YOLO` → `None` (run).
2. `tool.name in approved_tools` → `None`. (A session-approved call that cannot run still returns its normal error from `execute`.)
3. `level is AUTO_EDIT` and `tool.is_project_edit(arguments, context)` (exceptions count as `False`) → `None`.
4. `preview = tool.preview(arguments, context)` (exception → default preview with a note). `preview.error` → `ToolResult.error(preview.error)`: no question for a call that cannot run.
5. No prompter, or `prompter.available()` is false → refusal `unavailable` (below). The same call gets the same answer every time.
6. `answer = prompter.ask(ApprovalRequest(...))`; `APPROVE` → `None`; `APPROVE_SESSION` → add the tool name to the session set, `None`; `REJECT` → refusal `rejected`.

**Arguments are validated before previews run.** The loop calls the gate before `registry.execute`, which is where JSON Schema validation happens, so a model that sends `edit_file` without `old_string` would reach `preview` with a missing key. The gate therefore calls `validate_arguments(tool.parameters, dict(arguments))` (already used by the registry) first; if there are problems it returns `None` and lets `registry.execute` produce the usual `Invalid arguments` error, so no question is shown for malformed calls and `preview`/`is_project_edit` can assume the schema holds. This runs only on gate paths that would otherwise reach step 2 onward (not at `yolo`, not for tools without approval).

**Messages the model receives** (all `is_error=True`; `summary` is what the terminal shows):
| Case | `output` | `summary` |
|---|---|---|
| rejected, no reason | `The user rejected this <tool> call, so it was not run. Do not repeat the same call; propose a different approach or ask the user what they want.` | `rejected by user` |
| rejected with reason | same, plus ` The user said: "<reason>"` (reason cut at 500 characters) | `rejected by user` |
| cannot ask | `<tool> needs user approval, but nar cannot ask for it here (standard input and standard error must both be a terminal). It was not run. Tell the user what you wanted to do; they can allow it by starting nar with --approval auto-edit (file edits inside the project folder) or --approval yolo, or by setting approval in .nar/config.toml.` | `needs approval (cannot ask)` |
The "needs user approval" wording is kept from RRG-63 so scripted checks and the model's prior behavior carry over. The terminal shows the short `summary` (RRG-65's rule for error results that carry one) and the model gets the full text.

**Session memory is per tool name.** Chosen because it matches the wording of the issue ("yes for the whole session") and the user's mental model ("stop asking me about edits"), is trivial to explain in the prompt (`[a]lways allow edit_file this session`), and generalizes to MCP tools (`mcp__jira__create_issue`). Alternatives rejected: per exact call (the model rarely repeats an identical call, so it would never match and `a` would mean nothing); per command prefix or path glob (a product of its own, deferred as an allow-list); "everything for the session" (that is `--approval yolo`, chosen deliberately at startup, not by one keypress). Trade-off: `a` on `shell` approves every later command; the prompt label names the tool and the README says so, and the blocklist still applies.

**Approvals do not widen the level later:** `approved_tools` is not cleared when `level` changes (RRG-78), and a lower level (`ask` after `yolo`) still honors it; the user can restart. Accepted, documented.

### 4. `auto-edit` semantics and `is_project_edit`
`auto-edit` skips the question only when the tool says the call is a project edit. `files.is_project_path(path, cwd)` is `path.resolve().is_relative_to(cwd.resolve())` and not under `cwd/.git` or `cwd/.nar` (the resolved path is compared, so `..` and symlinks that leave the project are refused; on Windows the comparison is case-insensitive). `edit_file` and `write_file` return it for their resolved target.
- **Why the working-directory limit:** "edits files on its own" in the issue is about the user's project. A model that is prompt-injected by a web page should not silently write `C:\Users\me\.bashrc` or `~/.ssh/authorized_keys` because `auto-edit` is on.
- **Why `.git` and `.nar` are excluded:** both hold files that later run code or change permissions without another question (`.git/hooks/*`, `.nar/config.toml` can set `approval = "yolo"` or define hooks (RRG-72) and MCP servers (RRG-74)). Asking for those stays cheap because they are rarely edited. Other sensitive project files (`package.json` scripts, CI workflows, `Makefile`) are **not** excluded: a list that tries to be complete gives false confidence, and `ask` exists for users who want to see every edit.
- **Alternative rejected:** treating `shell` as auto-approved for read-only commands (`git status`, `ls`): requires parsing shell syntax; the existing blocklist already shows how fragile that is. Left to a future allow-list.
- `shell` is never a project edit (spec): the issue says `auto-edit` "asks before shell".

### 5. `edit_file` preview (`tools/edit_file.py`)
`execute` is split into a pure `_plan(arguments, context) -> EditPlan | ToolResult` (validation of `old_string`/`new_string`, `resolve_path`, `_load`, match counting, `new_text`, diff, counts) and the write. `execute` calls `_plan`, returns its error result as is, otherwise writes `plan.new_text` and builds the same `ToolResult` as today. `preview` calls `_plan`: on error `ToolPreview(title=f"Edit {shown}", error=message)`, otherwise `ToolPreview(title=f"Edit {shown} (+{added} -{removed})", diff=plan.diff)`. Because both paths run `_plan`, the preview diff and the result diff are identical (spec scenario "Preview equals the result"), and every error text exists once. No behavior of `execute` changes; the existing `edit_file` tests must pass unmodified (a task check).

### 6. `write_file` preview (`tools/write_file.py`)
`preview` reuses the same checks as `execute` in the same order (encoding, directory, blocking ancestor) and returns `ToolPreview(error=...)` for them; it never calls `mkdir` or `write_bytes`. Otherwise:
- **File exists, readable text** (size ≤ `MAX_FILE_BYTES`, no NUL in the first 8 KB, valid UTF-8): title `Overwrite <shown path>`, `diff` = unified diff from the current content to `content` (same `difflib` settings as `edit_file`; the helper `_unified_diff` moves to `files.py` so both tools share it). Identical content gives title `Overwrite <path> (no changes)` with an empty diff.
- **File exists but binary, too large or not UTF-8:** title `Overwrite <path>`, `detail` = `existing file cannot be shown as text; new content: N lines, M bytes` followed by the beginning of the content.
- **New file:** title `Create <path> (N lines)`, `detail` = the beginning of the content (the terminal cuts at `MAX_DETAIL_LINES`).
`shown path` is relative to the working directory when inside it, absolute otherwise (the `_shown_path` helper of `edit_file`, also moved to `files.py`), so a write outside the project is visibly different in the prompt.

### 7. `shell` preview (`tools/shell.py`)
`preview` runs the same pre-spawn checks as `execute` in the same order (empty command, `find_blocked`, no shell, missing cwd) and returns them as `ToolPreview.error`; otherwise `ToolPreview(title=f"Run in {shell.name} (timeout {timeout} s)", detail=command)`. The execution checks are factored into one helper `_refusal(command, context) -> str | None` used by both, so the messages cannot drift. Nothing is spawned. At `yolo` the preview is never computed and `execute` applies the same checks, as before.

### 8. Terminal prompter (`ui/approval_prompt.py`)
`TerminalPrompter(renderer: ToolRenderer, *, stdin=sys.stdin, read_line=input)`; the console is `renderer.console` (standard error). 
- `available()`: `stdin.isatty()` and `console.file.isatty()` (evaluated at every question, not cached; `FORCE_COLOR` does not make a pipe look like a terminal because the file object is asked, not Rich's `is_terminal`).
- `ask(request)` prints, on standard error: a rule-less header `Approval needed: <title>` (bold), the `detail` (dim, at most `MAX_DETAIL_LINES` lines, then `... N more lines not shown`), the `diff` through `ToolRenderer.show_diff` (same colors as the result diff; at most `MAX_DIFF_LINES` lines, then the same note), and, when anything was cut, `(preview shortened)` before the question. All text is printed with `Text(...)`/`markup=False`, never interpreted (spec: `[red]` literal). Then it prints the question `Allow <tool>? [y]es / [n]o / [a]lways allow <tool> this session: ` without a newline and calls `read_line("")` (the prompt text goes through the Rich console so it lands on standard error; `input("")` then reads standard input and prints nothing to standard output).
- **Answer parsing:** strip, lowercase; `y`/`yes` → approve; `a`/`always` → approve for session; `n`/`no` → reject then a second line `Reason (optional, Enter to skip): `; anything else (including empty) re-asks with `Please answer y, n or a.` There is no default on Enter, so a stray keypress never approves.
- **End of input** (`EOFError`) at the question or at the reason line → reject without a reason (for the reason line: reject with the reason typed so far, i.e. none). `KeyboardInterrupt` is **not** caught: it cancels the request through the path in Context.
- After `a` it prints a dim confirmation `edit_file will not ask again in this session`.
- Windows: `input()` handles Ctrl+Z+Enter as EOF and Ctrl+C as `KeyboardInterrupt`; no raw-mode key reading is used (a single-keypress prompt would need `msvcrt`/`termios` branches and breaks piped test input), so the user types the letter and presses Enter. Documented trade-off.
- Spinner: nothing to do (Context). The prompter does not touch `AgentView`.

**Alternatives rejected:** `prompt_toolkit` for the question (already planned for slash commands in RRG-78, but importing it here couples two changes and the prompt needs no completion); `rich.prompt.Prompt.ask` (writes the prompt to the console correctly, but its re-ask text, choices rendering and EOF behavior are less controllable than 15 lines of code, and it reads with the same `input()`); a Textual dialog (new dependency, out of scope).

### 9. Non-interactive and headless behavior
`available()` false → `refusal unavailable` (Decision 3), identical for every call, no waiting, no partial output. Cases: `echo x | nar "..."` (stdin not a TTY), `nar "..." 2> log` (the user could not see the question), CI. Single-mode `nar "prompt"` in a terminal is interactive for approval purposes: it asks, then exits when the request ends. Subagents (RRG-73) and any other `Agent` built with `approval_gate(level)` and no prompter get the same refusal, which is the safe default for background work. A request whose every approval-required call is refused still completes normally (exit code 0): the model is told and can answer; failing the run would hide a useful answer. Revisit if CI users ask for a non-zero code.

### 10. CLI wiring (`cli/main.py`, `cli/runner.py`)
- `main.py`: `@click.option("--approval", type=click.Choice([level.value for level in ApprovalLevel]), help="Autonomy level for this run: ask, auto-edit or yolo (overrides the configuration).")`; `load_config(cwd, overrides={"provider": provider, "model": model, "approval": approval})`. The loader already labels the source `command line` and validates; `--show-config` needs no change. Click's own error lists the allowed values for an invalid option.
- `runner.create_agent`: `registry = create_default_registry(cwd)`; `prompter = TerminalPrompter(view.tools)`; `gate = approval_gate(config.approval, prompter, registry.context)`; `Agent(..., gate=gate)`. The same `view` object is used for both modes, so interactive and single mode behave identically. `registry.context` is a new read-only property so the gate previews with the same working directory the tools use, with no second copy of `cwd`.
- `repl.py` and the loop: unchanged.

### 11. Tests
Offline `pytest`; a `ScriptedPrompter` (list of answers, records the requests, configurable `available()`) replaces the terminal in gate and loop tests.
- `test_approval_gate.py`: all level × tool-kind combinations (`ask` asks; `auto-edit` skips only project edits, still asks for shell, for outside paths, for `.git`/`.nar` and for a plain tool; `yolo` never asks and never previews), `approve`, `reject` with and without reason (message text, `summary`, reason cut at 500), `APPROVE_SESSION` then a second call not asked and another tool asked, `preview.error` returned without asking, preview raising falls back to the default preview, `is_project_edit` raising counts as false, unvalidated arguments return `None` so the registry reports them, no prompter and `available() == False` give the same refusal and never call `ask`, changing `gate.level` takes effect, a tool registered later with only `requires_approval = True` is asked with its arguments as preview.
- `test_approval_prompt.py`: `TerminalPrompter` with `Console(file=StringIO())` and an injected `read_line`: answers `y/yes/Y`, `n` + reason, `n` + Enter, `a`, junk then `y` (re-asked), empty then `n`, `EOFError` at the question and at the reason line, `KeyboardInterrupt` propagates, `available()` for a TTY and a non-TTY stdin/console stand-in, long detail and long diff are cut with the note and `(preview shortened)`, `[red]` in the command prints literally, nothing is written to the standard output stand-in.
- Tool tests: `test_tools_base.py` (default preview is name + JSON arguments with non-ASCII intact, default `is_project_edit` false), `test_tools_edit_file.py` (preview diff equals result diff on the same file, file unchanged after preview by bytes and mtime, every `execute` error appears as `preview.error` with the same text, CRLF file, `is_project_edit` for relative, `..`, absolute-outside and `.git/x`), `test_tools_write_file.py` (new file, overwrite diff, identical content, binary existing file, directory target, blocking ancestor, no parent folder created by preview, outside path), `test_tools_shell.py` (preview shows command, shell name and default/explicit timeout; blocked and empty commands give `error`; `Popen` monkeypatched to fail if called).
- `test_agent_loop.py`: the two existing gate tests move to the new `approval_gate` import; new loop-level tests with a scripted provider and prompter: reject then the model answers (the "done when": file intact, request continues), two calls in one answer (reject, approve) keep order and one result each, `KeyboardInterrupt` from the prompter rolls the request back, an approve-for-session call in request 1 is not asked in request 2 on the same agent.
- `test_cli.py`: `--approval yolo --show-config` shows `(command line)`, `--approval always` fails listing the three levels, `--help` lists `--approval`, a full single-mode run with the fake provider where stdin is not a TTY gets the "cannot ask" refusal and exits 0 with the answer on stdout and nothing on stdout from the prompt.
- `test_tool_view.py`: `show_diff` prints diff colors and literal brackets (the existing diff tests keep passing). `test_tools_registry.py`: `registry.context` is the context passed in; the `approval_gate` test of RRG-65 is updated (`ASK` without a prompter refuses with "needs user approval", `YOLO` returns `None`).
- Manual checklist: `tasks.md` last task.

### 12. README
`README.md`: a section "Approvals": the three levels with one example each, the prompt with `y` / `n` / `a`, the optional reason, what is shown for each built-in tool, `auto-edit` limits (working directory only, `.git` and `.nar` always asked), the `--approval` option, non-interactive behavior and how to allow it, session approval and the `shell` caveat. It also removes the sentences that say the agent only runs `shell`, `write_file` and `edit_file` at `yolo` (in "How the agent works" and the `shell` subsection) and the "until RRG-69" notes.

## Risks / Trade-offs

- [The file or the environment changes between the preview and the execution (another process, or the user editing in an editor while the question is open), so the approved diff is not exactly what is applied] → The window is the time the user takes to answer; `edit_file` still requires the unique match at execution and fails safe, `write_file` shows the full replacement it will write. Not solved here (non-goal); a content hash check in `execute` would be the follow-up.
- [A shortened preview hides part of a very long command or diff, and the user approves what they did not see] → The cut is announced twice (note and `(preview shortened)`); the limits (60 detail lines, 100 diff lines) are larger than normal commands and edits; a command is shown whole up to those limits. Rejecting is one keypress, and the model can resend smaller changes.
- [`a` on `shell` approves every future command of the session, including ones the user would reject] → Prompt label names the tool; README warns; the blocklist still applies; the user who wants per-command control answers `y`.
- [The model learns that `n` produces a long instruction and keeps proposing variations, wasting turns] → `max_turns` bounds it; the message tells it to ask the user. Rejection reasons help the model change course.
- [Typing a letter plus Enter is slower than a single keypress] → Deliberate (Decision 8); keeps one code path on Windows, Linux and macOS and makes the prompt testable with plain `input`.
- [A prompt-injected model writes a file at `auto-edit` that later runs code (`package.json` scripts, `Makefile`, CI files)] → Only `.git` and `.nar` are special-cased (Decision 4); `ask` remains the default and the README says what `auto-edit` trusts.
- [`.nar/config.toml` is edited by a human through the model on purpose and the user is now asked every time at `auto-edit`] → Acceptable: rare, and one `y`.
- [Preview and execution drift for a tool whose author writes two separate code paths] → The three built-in tools share one helper each (`_plan`, `_refusal`, shared checks) and tests compare preview and result; MCP tools use the default preview that cannot drift.
- [Windows consoles using a legacy code page render some characters of a diff as `?`] → Reuse of the existing lossy-encoding handling in `ToolRenderer` for diff lines.

## Migration Plan

No data migration. `approval = "yolo"` in existing configuration files behaves as before; files with `ask` (the default) change from "refuse approval-required tools" to "ask", and non-interactive runs at `ask` keep refusing (with a clearer message). `approval_gate` moves from `nar.agent.loop` to `nar.agent.approval`; `nar.agent` still exports it. Rollback: revert the PRs; the agent returns to RRG-63's refuse-unless-`yolo` behavior and the tools lose their optional `preview` methods without other effects.

## Open Questions

- Should a non-interactive run whose approval-required calls were all refused exit with a non-zero code? Not needed for the "done when"; Decision 9 keeps exit 0 and can change without touching the specs of this capability.
- Should `auto-edit` also auto-approve a short, fixed set of read-only `shell` commands (`git status`, `git diff`, `ls`)? Deferred together with allow-lists; no change to the spec text above would be needed beyond a new requirement.
