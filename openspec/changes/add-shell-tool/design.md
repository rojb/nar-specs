# Design

## Context

After RRG-62, RRG-64, RRG-66 and the agent loop (RRG-63) the `nar` package has: `src/nar/tools/` (`Tool`, `ToolContext`, `ToolResult` with `output`, `is_error`, `summary`, `diff` in `base.py`; `ToolRegistry` in `registry.py`; `limits.py` with `MAX_OUTPUT_CHARS = 30_000`; `files.py`; the file and search tools; `register_builtin_tools()` / `create_default_registry()` in `builtin.py`), `src/nar/ui/tool_view.py` (`ToolRenderer`), `src/nar/agent/loop.py` (`Agent`, `approval_gate`) and the runner `scripts/run_tool.py`. Motivation and scope: see `proposal.md`. Required behavior: see `specs/shell-tool` and the `tool-framework` delta (tool activity display).

Facts that shape the design (checked in the code and on the development machine, Windows 11 with Windows PowerShell 5.1, Git Bash, no `pwsh`):
- `ToolRegistry.execute` is the single choke point: it validates arguments (JSON Schema, so `minimum`/`maximum` give the "names `timeout` and its range" error for free), catches `Exception` and truncates `output` to 30,000 characters keeping only the **head**. It does not catch `KeyboardInterrupt`, so an interrupt raised inside a tool propagates to `run_request` (`agent/loop.py` and `cli/runner.py`), which closes the generator, rolls the request back and lets the REPL print "cancelled". The shell tool only has to clean up its child before re-raising.
- `approval_gate(level)` in `agent/loop.py` refuses tools with `requires_approval` unless `approval = "yolo"`, until RRG-69. It runs before `registry.execute`, so it needs no change; the blocklist runs inside the tool and therefore also at `yolo`.
- Error results are printed in full by `ToolRenderer.show_result` (`✗ <message>`), which is wrong for a timeout result that carries up to ~28,000 characters of partial output (Decision 8).
- Windows PowerShell 5.1 with `-Command <string>` loses the exit code of a native command (`python -c "import sys; sys.exit(3)"` exits with 1, measured), prints PowerShell progress as XML on a redirected stderr unless `$ProgressPreference` is silenced, and encodes its output with the console's OEM code page. `-EncodedCommand` was measured too and rejected (Decision 2). `taskkill /PID <pid> /T /F` kills `powershell.exe` and the `python.exe` it started, and a second `communicate()` afterwards still returns the output printed before the kill (measured). A long blocking wait can delay the delivery of Ctrl+C on Windows, so the tool waits in short polling slices instead.
- The agent's own Ctrl+C: a console Ctrl+C is delivered to every process attached to the console, so without isolation the shell child gets the signal at the same time as `nar` and may exit on its own (or ignore it); the tool must own the child's lifetime either way.

## Goals / Non-Goals

**Goals:**
- One tool, `shell`, with the contract of RRG-65: command, timeout, stdout/stderr/exit code, bounded output, platform shell, blocklist, `requires_approval`.
- The tool can never hang `nar` (stdin closed, timeout, tree kill) nor exhaust its memory (bounded capture).
- The same observable behavior on Windows and POSIX; every platform-specific line lives in one module and is tested on its own platform.
- No new dependency and no new configuration key.

**Non-Goals:**
- A sandbox. The blocklist is a regex safety net; RRG-69 approval is the control. Obfuscated commands (`r""m -rf /`, base64, variables, `bash -c "$(...)"`) are not caught.
- Persistent shell sessions: each call starts a fresh shell, so `cd` and variables do not carry over (the tool description tells the model to chain commands or use full paths).
- Streaming output while the command runs, background jobs, interactive programs, pseudo-terminals, resource limits (CPU, memory) and network control.
- A configurable shell, timeout or blocklist (`.nar/config.toml` keys). Revisit if users need them; the constants are in one place.
- Killing processes that a command deliberately detached (`Start-Process`, `nohup ... &`, `setsid`): they survive on exit of the command; only the tree alive at timeout or Ctrl+C is killed.
- Conversion of output encodings other than UTF-8 (see Risks).

## Decisions

### 1. Module layout
```
src/nar/
├── tools/
│   ├── shell.py             NEW       ShellTool (schema, flow, result formatting), constants
│   ├── shell_process.py     NEW       find_shell, build_argv, run_command, kill_tree, StreamCapture
│   ├── shell_blocklist.py   NEW       find_blocked(command) -> reason | None, PATTERNS
│   ├── builtin.py           MODIFIED  register_builtin_tools(): + ShellTool (appended last)
│   ├── base.py, registry.py, limits.py, files.py   unchanged
│   └── __init__.py          unchanged (built-ins are not re-exported)
└── ui/
    └── tool_view.py         MODIFIED  ToolRenderer.show_result: error branch prints result.summary or result.output
scripts/run_tool.py          unchanged (prints result.output after the rendered result)
pyproject.toml               unchanged
tests/test_tools_shell_blocklist.py   NEW
tests/test_tools_shell_process.py     NEW
tests/test_tools_shell.py             NEW
tests/test_tool_view.py               MODIFIED  error with summary
tests/test_tools_registry.py          MODIFIED  default registry lists `shell`, flags, schema
README.md                             MODIFIED  `shell` row, section on the tool
```
`nar/agent/*`, `nar/cli/*`, `nar/config/*`, `nar/providers/*`, `nar/errors.py` are **not** touched (the agent loop already passes registry definitions to the model and already gates `requires_approval`). **New config keys: none** (neither `~/.nar/config.toml` nor `.nar/config.toml`). Constants in `shell.py`: `DEFAULT_TIMEOUT = 120`, `MAX_TIMEOUT = 600`, `STREAM_HEAD_BYTES = 4_000`, `STREAM_TAIL_BYTES = 10_000`.

Why three modules: the blocklist is pure text (fast tests, no process); `shell_process` is the only code that touches `subprocess`, signals and platform branches and is testable with a fake `which`; `shell.py` stays a thin tool that wires them. The file names keep the `shell` prefix so they sort together and are not mistaken for general helpers.

### 2. Shell selection and invocation (`shell_process.find_shell`, `build_argv`)
```python
@dataclass(frozen=True)
class ShellSpec:
    name: str            # "PowerShell 7", "Windows PowerShell", "bash", "sh"
    executable: str      # absolute path from shutil.which
    powershell: bool

def find_shell(*, windows: bool | None = None, which=shutil.which) -> ShellSpec | None
def build_argv(shell: ShellSpec, command: str) -> list[str]
```
- **Windows:** `which("pwsh")` first, then `which("powershell")`. Argv: `[exe, "-NoProfile", "-NonInteractive", "-Command", script]`. `cmd.exe` is not used and `bash` is never used on Windows even when Git Bash is on `PATH` (the user's terminal on Windows is PowerShell, and `C:\Windows\System32\bash.exe` on this machine is the WSL launcher).
- **POSIX:** `which("bash")`, then `which("sh")`. Argv: `[exe, "-c", command]` (non-login, non-interactive: no profile, no rc files).
- `script` for PowerShell is `PREFIX + command + SUFFIX`:
  - `PREFIX = "$ProgressPreference = 'SilentlyContinue'; [Console]::OutputEncoding = [System.Text.UTF8Encoding]::new($false); $OutputEncoding = [Console]::OutputEncoding\n"` – silences progress output and makes the shell's own output (cmdlets, `Write-Output`) UTF-8 with no BOM, which is what the tool decodes.
  - `SUFFIX = "\nif (-not $?) { if ($LASTEXITCODE) { exit $LASTEXITCODE } else { exit 1 } }"` – restores the exit code of the last native command (`python -c "import sys; sys.exit(3)"` then exits with 3 on Windows PowerShell 5.1; on PowerShell 7 it is a harmless no-op). A failing cmdlet exits with 1, `exit 7` exits with 7.
  - The command goes on its own line so a trailing `# comment` cannot swallow the suffix, and its text is passed unmodified.
- **Alternatives rejected:**
  - `-EncodedCommand` (base64 of UTF-16LE): immune to quoting, but PowerShell then reports its own errors as CLIXML on stderr (`#< CLIXML <Objs ...`), unreadable for the model (measured). With `-Command` errors are plain text, and quoting problems exist only in the rare "double quotes inside a double-quoted native argument" case, which behaves exactly as in the user's own terminal.
  - Feeding the script through standard input (`-Command -`): conflicts with "standard input closed".
  - `pwsh -File temp.ps1`: temp-file lifecycle and execution-policy issues for no gain.
  - Preferring Windows PowerShell over `pwsh`: `pwsh` supports `&&`/`||` and has consistent exit codes; the model gets the better shell when the user has installed it. The chosen shell's name is in the tool description, so the model can adapt syntax (for instance not to use `&&` when only Windows PowerShell 5.1 is present).
  - A `shell` config key (`"powershell" | "cmd" | "bash"`): not requested by RRG-65; an extra surface that the approval change (RRG-69) would have to explain. Revisit if someone needs `cmd`.
- `find_shell` is called once, in `ShellTool.__init__`, so the description can name the shell; if it returns `None` the tool still registers and `execute` returns the "no shell found" error. `windows` defaults to `sys.platform == "win32"`; both parameters exist for tests.

### 3. The tool contract (`shell.py`, `ShellTool`)
```python
parameters = {
    "type": "object",
    "properties": {
        "command": {"type": "string", "minLength": 1, "description": "The command line to run, e.g. 'python --version'."},
        "timeout": {"type": "integer", "minimum": 1, "maximum": 600,
                    "description": "Seconds to wait before killing the command (default 120)."},
    },
    "required": ["command"],
    "additionalProperties": False,
}
requires_approval = True
```
`description` is built in `__init__` from the selected shell, for example: "Run a command line in PowerShell (Windows) and return its exit code, standard output and standard error. Each call starts a fresh shell in the working directory (cd and variables do not carry over). Standard input is closed, so interactive commands cannot work. Output is limited and long output keeps its beginning and end. Commands are stopped after `timeout` seconds (default 120, at most 600). Some destructive commands are refused. Use ';' to chain statements." For bash the syntax hint says `&&`. The class sets no class-level `description`; `Tool` only requires the attribute to exist on the instance.

Flow of `execute(arguments, context)`; every failure is a `ToolResult` and nothing raises:
1. `command.strip()` empty → `ToolResult.error("command must not be empty")` (the schema rejects the empty string; this covers `"   "`).
2. `find_blocked(command)` → `ToolResult.error(f"command blocked by nar's safety list ({reason}). Do not retry it or work around it; tell the user what you wanted to do so they can run it themselves.")`. Runs before anything else touches the system.
3. `shell is None` → `ToolResult.error("no shell found: install PowerShell (Windows) or bash/sh (Linux, macOS)")`.
4. `not context.cwd.is_dir()` → `ToolResult.error(f"working directory not found: {cwd}")`.
5. `run_command(build_argv(...), cwd, timeout)`; `OSError` on spawn → `ToolResult.error(f"cannot start {shell.name}: {exc.strerror or exc}")`.
6. Build the result (Decision 6) and the summary (Decision 7).

### 4. Running the process and bounding memory (`shell_process.run_command`)
```python
@dataclass(frozen=True)
class RunResult:
    exit_code: int | None      # None when the command was killed
    stdout: StreamCapture
    stderr: StreamCapture
    timed_out: bool

def run_command(argv: list[str], cwd: Path, timeout: float) -> RunResult
```
- `Popen(argv, cwd=cwd, stdin=subprocess.DEVNULL, stdout=PIPE, stderr=PIPE)`; the environment is inherited (`env=None`). Isolation of the child's process tree: Windows `creationflags=subprocess.CREATE_NEW_PROCESS_GROUP` (the child no longer receives the console's Ctrl+C, so only `nar` reacts and then kills the tree on purpose); POSIX `start_new_session=True` (the child leads a new session and process group, which is what `os.killpg` needs; it also detaches it from the terminal's Ctrl+C).
- **Capture:** one daemon thread per stream runs `StreamCapture.drain(pipe)`, a loop of `pipe.read1(65536)` until EOF. `StreamCapture` keeps the first `STREAM_HEAD_BYTES` (4,000) in `head`, the last `STREAM_TAIL_BYTES` (10,000) in `tail` (a `bytearray` trimmed after each chunk) and counters: `total` bytes, `lines` (newlines seen, plus one when the stream is non-empty and does not end in a newline). Memory per stream is therefore bounded by about 14 KB plus one 64 KB chunk, however much the command prints. `text()` decodes `head` and `tail` with `errors="replace"`, joins them with `\n[... N bytes omitted ...]\n` when `total` exceeds `head + tail`, and replaces `\r\n` with `\n`. A multi-byte character cut at a boundary becomes one `U+FFFD`; accepted.
- **Wait and timeout:** the main thread polls `proc.poll()` every 0.05 s up to `deadline = time.monotonic() + timeout`. Polling instead of `communicate(timeout)`/`wait(timeout)` keeps the wait interruptible by Ctrl+C on Windows, where a blocking wait is delivered late. 50 ms latency is irrelevant for commands that take at least a process start (~100-500 ms for PowerShell).
- On expiry: `timed_out = True`, `kill_tree(proc)`, then `proc.wait()`.
- On `BaseException` while waiting (`KeyboardInterrupt` is the case that matters): `kill_tree(proc)`, a short best-effort `proc.wait(timeout=5)` and re-raise. Nothing is returned; the registry does not catch `KeyboardInterrupt`, so the interrupt reaches `run_request` as for any cancelled request.
- After the process ends, each reader thread is joined with a 2 s limit. A command that left a background process holding the pipes open (for example `Start-Process`, `cmd /c start`, `nohup x &` without redirection) would otherwise keep the readers waiting for EOF forever; after the limit the tool returns what it has and leaves the daemon threads and the background process alone. Pipes are not closed from the main thread (closing a handle that another thread is blocked reading can hang on Windows).
- **`kill_tree(proc)`:** Windows: `subprocess.run(["taskkill", "/PID", str(proc.pid), "/T", "/F"], capture_output=True, timeout=10)` (its localized output is ignored), then `proc.kill()` as a fallback, all exceptions swallowed; POSIX: `os.killpg(proc.pid, signal.SIGKILL)` with `ProcessLookupError`/`PermissionError` ignored, then `proc.kill()`. Processes that already re-parented away from the tree (double fork, `setsid`) are not reachable; accepted.
- **Alternatives rejected:** a Windows job object with `KILL_ON_JOB_CLOSE` through `ctypes` (the only way to kill descendants that outlive their parent, about 60 lines of `ctypes` for a case the proposal puts out of scope); `psutil` (new dependency); `subprocess.run(timeout=...)` (it kills only the direct child, so on Windows `powershell.exe` dies while the `python.exe` it started keeps running, and it gives no control over the pipes' memory use); SIGTERM-then-SIGKILL on POSIX (a command that traps SIGTERM would delay the timeout; the process is meant to die).

### 5. Dangerous command blocklist (`shell_blocklist.py`)
`find_blocked(command: str) -> str | None` returns the human reason of the first matching pattern, or `None`.

**Normalization** (before matching): replace every run of line breaks with `;` (a multi-line command is several commands), lower-case, delete every `'` and `"` (so `rm -rf '/'` and `r""m` style quoting around arguments cannot hide a target), and collapse whitespace to single spaces. The same list is used on every platform and for both shells (a model on Windows may write `rm -rf /` because PowerShell aliases `rm`, and a model on Linux may write `shutdown`).

**Shared fragments** (regex, applied to the normalized text, `re.search`):
```
END     = (?=[\s;&|)]|$)                                  # the target ends here
HOME    = (?:~|\$home|\$\{home\}|\$env:(?:userprofile|homepath))
TARGET  = (?:HOME[\\/]?\*?|/\*?|[a-z]:[\\/]?\*?|\$env:(?:systemroot|windir|systemdrive)[\\/]?\*?)
CMDPOS  = (?:^|[;&|(]\s*|\bsudo\s+)                       # start of a command
```
`TARGET` is deliberately only the root itself or the home folder, optionally followed by `/`, `\` or `*`: `rm -rf /tmp/x`, `rm -rf ~/proj/build` and `del /s /q C:\proj\tmp\*.log` are not blocked.

| # | Reason shown to the model | Pattern |
|---|---|---|
| 1 | recursive removal of the filesystem root or the home folder | `\brm\s+(?:-\S+\s+)*?(?:-\S*r\S*\|--recursive)\s+(?:-\S+\s+)*TARGET END` |
| 2 | recursive removal of a drive root, the system folder or the home folder (PowerShell) | `\b(?:remove-item\|ri\|del\|erase\|rmdir\|rd)\b(?=[^;&\|]*\s-r(?:ecurse)?\b)[^;&\|]*?\s TARGET END` |
| 3 | recursive removal of a drive (cmd) | `\b(?:del\|erase\|rmdir\|rd)\b(?=[^;&\|]*\s/s\b)[^;&\|]*?\s[a-z]:\\?\*?(?:\.\*)?END` |
| 4 | formatting a disk | `CMDPOS format(?:\.com)?\s+[a-z]:` or `\bformat-volume\b` or `\bclear-disk\b` or `\bdiskpart\b` |
| 5 | creating a filesystem | `\bmkfs(?:\.\w+)?\b` |
| 6 | writing to a disk device | `\bdd\b[^;&\|]*\bof=/dev/` or `>\s*/dev/(?:sd\|hd\|vd\|nvme\|disk)` |
| 7 | fork bomb | `:\(\)\s*\{\s*:\s*\|\s*:\s*&\s*\}\s*;\s*:` |
| 8 | shutting down or rebooting the machine | `CMDPOS (?:shutdown\|reboot\|poweroff\|halt\|init\s+[06]\|stop-computer\|restart-computer)\b` |

(`\|` above is a literal `|` inside a markdown table; the code has plain `|`.) The first three rows stop at command separators (`[^;&|]*`), so `echo done; rm -rf /` is caught on the `rm` segment and `rm -rf build/ && ls /` is not. Rows 4 and 8 are anchored to a command position, so prose such as `echo "do not shutdown"`, `git commit -m 'fix shutdown hook'` or `git log --format=%H` is not blocked.

The patterns were exercised against about 40 positive and 25 negative commands before being written here (`rm -rf /`, `RM   -fr   ~/`, `sudo rm -rf /*`, `rm --recursive --force /`, `rm -rf $HOME`, `Remove-Item -Recurse -Force C:\`, `Remove-Item C:\ -Recurse -Force`, `ri -r -fo C:\`, `del /f /s /q C:\*.*`, `rd /s /q C:\`, `format c: /q`, `mkfs.ext4 /dev/sda1`, `dd if=/dev/zero of=/dev/sda`, `:(){ :|:& };:`, `ls && shutdown -h now`, `(shutdown -h now)`, ...; and `rm -rf build/`, `rm -rf /tmp/x`, `Remove-Item -Recurse -Force .\build`, `Get-ChildItem C:\ -Recurse`, `Format-Table`, `dd if=a of=b`, ...). Those lists become the parametrized cases of `tests/test_tools_shell_blocklist.py`. Known false positive: removing the quotes makes `python -c "print('format c:')"` look like a command position (`print(format c:`); a blocked harmless command is acceptable for a safety net and the error tells the model to ask the user.

**What it does not claim:** the list is a safety net for model mistakes. It cannot see through variables, encoded payloads, scripts the model wrote earlier (`bash cleanup.sh`), `find / -delete`, `git clean -fdx` at the wrong folder, `curl ... | sh` or destructive commands that are not on the list. The README and the tool description say so; the real control is approval (RRG-69). Alternatives: a tokenizer-based parser (`shlex` for POSIX, a PowerShell parser for Windows) would be more exact and far larger; an allow-list of commands would defeat the purpose of a general shell tool.

### 6. Result format and error policy
```
exit code: 0
stdout:
Python 3.11.9
```
```
exit code: 2
stdout:
<text>
stderr:
<text>
```
Rules: the first line is always `exit code: N`; a `stdout:` / `stderr:` section appears only when that stream has bytes (the label is on its own line, the text follows with its trailing newline trimmed); when both are empty the second line is `(no output)`. Streams are separate because the model reads stderr differently from stdout (warnings versus data), and because the per-stream cap of Decision 4 needs them apart.

| Situation | `is_error` | `output` | `summary` (terminal) |
|---|---|---|---|
| Ran, exit code 0 | false | report above | `exit 0 · 1 line` / `exit 0 · 12 lines` / `exit 0 · no output` |
| Ran, exit code ≠ 0 | **false** | report above | `exit 3 · 2 lines` |
| Timed out (killed) | **true** | `command timed out after N seconds and was killed` + the two sections | `timed out after N s · 2 lines` |
| Blocked, empty command, no shell, bad cwd, cannot start | true | one-line message | none (message shown) |

**Why a non-zero exit code is not a tool error.** The command did what it was asked and the result is information: `pytest` exits 1 when tests fail, `grep` exits 1 when nothing matches, `git diff --exit-code` uses 1 for "differences". Provider adapters prepend an error marker to error results; marking `pytest` failures that way would push the model to treat a normal test failure as "the tool broke" and retry the same call. The exit code is always the first line, so no information is lost, and the spec requires it. **Cost:** the terminal prints the green success indicator for a command that exited 3; the summary `exit 3 · 2 lines` is the cue. A distinct indicator would need a new `ToolResult` field and is left to a later UI change (see Open Questions).

**Why a timeout is an error.** The command did not complete, so the model cannot trust the result as the command's answer, and the terminal should show it as a failure. The partial output is still in `output` (the spec requires it), and the new `summary` on an error keeps the terminal short (Decision 8). A timeout does not set an exit code: `exit_code` is `None` and the text says "killed".

The report is built from at most 14,000 characters per stream plus short labels, so it is below 30,000 characters and the registry's head-only truncation never cuts the tail the tool worked to keep. `run_tool.py` prints `result.output` and exits 0 for a non-zero command exit, 1 for an error result.

### 7. Summary line
`summary = f"exit {code} · {n} line(s)"`, where `n = stdout.lines + stderr.lines` counted over the whole stream (including omitted bytes), singular for 1, and `exit {code} · no output` when both streams are empty. For a timeout: `timed out after {timeout} s · {n} line(s)`. The middle dot is U+00B7, present in cp1252, cp850 and cp437, so it prints on Windows consoles without the replace logic that `_print_diff` needs; the indicator logic in `_indicator` is unchanged.

### 8. Terminal display of error summaries (`ui/tool_view.py`, tool-framework delta)
`ToolRenderer.show_result` error branch becomes `message = result.summary or result.output` and prints `✗ <message>` as today. Every existing error result has `summary=None`, so nothing changes for them (a regression test pins this). Only `shell` sets a summary on an error. The model still receives `result.output` unchanged. **Alternative rejected:** truncating every error message in the terminal at N lines: it would also hide the validation errors the user sometimes needs to read in full.

### 9. Cancellation (Ctrl+C)
`run_command` handles the interrupt as in Decision 4: kill the tree, re-raise. With isolation (`CREATE_NEW_PROCESS_GROUP` / new session) the child never receives the console's Ctrl+C itself, so there is exactly one actor, `nar`, and the sequence is deterministic: Python raises `KeyboardInterrupt` in the poll loop → `kill_tree` → the exception propagates through `ToolRegistry.execute` (it only catches `Exception`) → `Agent.run`'s `finally` rolls the request back → `run_request` re-raises → the REPL prints "cancelled". No change in `agent/` is needed.
- Automated test: a fake sleep that raises `KeyboardInterrupt` once a real child (a Python process waiting 60 s) is running; assert the exception propagates and the child (and its own child) are gone. The real keypress is checked by hand in the final task.
- One Windows detail: `CREATE_NEW_PROCESS_GROUP` also disables Ctrl+C handling in the child, which is what we want; the child cannot be stopped by the user other than through `nar`.

### 10. Registration and the approval gate
`register_builtin_tools` registers `ShellTool()` **last** (after `GrepTool`): the export order is the registration order, appending avoids touching the relative order the other changes assert and keeps this change independent of `add-search-tools` bookkeeping. `tests/test_tools_registry.py` is updated: names end with `"shell"`, `required["shell"] == ["command"]`, approval map has `"shell": True`. `ShellTool` is instantiated per `create_default_registry()` call, so shell lookup happens at registry creation, not at import time.

Approval: `Tool.requires_approval = True` is all the framework needs. With RRG-63's gate and `approval` below `yolo`, the model receives "shell needs user approval ..." and the command does not run; manual verification therefore sets `approval = "yolo"`. When RRG-69 arrives it shows `arguments["command"]` (and `timeout`) before running; the blocklist still applies after an approval.

### 11. Tests
Offline `pytest`; process tests run real shells, so they are slower (PowerShell starts in 0.3-1 s on Windows). Cross-platform commands come from a helper in the test module that writes a small Python script to `tmp_path` and builds the command for the active shell: PowerShell `& '<sys.executable>' '<script>'`, POSIX `'<sys.executable>' '<script>'`; scripts print, write to stderr, exit with a code, sleep or spawn a child. Platform-specific cases use `skipif(sys.platform == "win32")` / `skipif(sys.platform != "win32")`, and cases that need a real shell are skipped when `find_shell()` is `None`.
- `test_tools_shell_blocklist.py` (no processes): parametrized blocked commands (every example in the spec plus the list in Decision 5), parametrized allowed commands, normalization (case, tabs, repeated spaces, quotes, newline as separator), `echo done; rm -rf /` blocked, reason text present.
- `test_tools_shell_process.py`: `find_shell` with fake `which` for the four combinations and for "nothing found" (`windows=True/False`); `build_argv` for PowerShell (flags `-NoProfile -NonInteractive -Command`, prefix, command on its own line, suffix) and bash/sh (`-c`); `StreamCapture` unit cases (under limit untouched, over limit keeps head and tail with the omitted count, line counts with and without a trailing newline, a multi-byte character split at the edge does not raise); real runs: exit 0 with output, exit 3 (PowerShell suffix keeps the code), stderr separate, stdin closed (a script reading `sys.stdin.read()` returns at once with empty input), timeout returns about when expected with partial output, the timed-out child's own child stops (the grandchild appends to a file every 0.1 s; the file stops growing after the call returns), an endless printer is bounded and times out, KeyboardInterrupt cleanup, `Popen` `OSError` from a bogus executable.
- `test_tools_shell.py`: contract (name, schema, `requires_approval`, description names the shell), blocked → error and `Popen` never called (monkeypatched), empty command, missing cwd, no shell, report format cases (first line, sections, `(no output)`), summaries (`exit 0 · 1 line`, plural, `no output`, timeout summary), non-zero exit is not an error, timeout is an error with partial output and summary, `\r\n` normalized, invalid UTF-8 byte replaced, through `ToolRegistry.execute`: `timeout` 0 and 601 rejected naming `timeout`, `command` required; `python --version` run for real compares to `sys.version`-derived text.
- `test_tool_view.py`: error with a summary prints the summary and not the message; error without one prints the message (regression).
- `test_tools_registry.py`: names, schema `required`, approval flag, export shape for `shell`.
- Manual checklist in `tasks.md`: `scripts/run_tool.py` cases and the agent run of the issue's "done when".

### 12. README
`README.md` gets a `shell` row in the built-in tools table (approval: yes), and a short subsection: which shell is used on each OS, the timeout (default 120 s, max 600 s), output limits, that a non-zero exit code is a normal result, that the blocklist is a safety net and not a sandbox, that the agent runs it only with `approval = "yolo"` until RRG-69, and an example `run_tool.py shell` call. It also fixes the sentence in "How the agent works" that says none of the current built-in tools need approval (`write_file`, `edit_file` and `shell` do).

## Risks / Trade-offs

- [The blocklist can be bypassed (obfuscation, scripts, commands not on the list) and gives a false sense of safety] → Stated in the spec, tool description, README and this design; `requires_approval` is set so RRG-69 can show every command before it runs; until then `yolo` is an explicit user choice.
- [Windows PowerShell 5.1 output encoding: native programs write in their own code page (Python on a pipe uses cp1252, many legacy tools use OEM cp850/437), while the prefix makes PowerShell decode as UTF-8, so accented characters from such programs can appear as U+FFFD] → ASCII output (versions, test results, git) is unaffected; the report never raises. Possible follow-up: set `PYTHONIOENCODING=utf-8` for Python children. Not done here because "environment inherited" is part of the spec and the change would be a silent behavior difference between nar and the user's terminal.
- [A command that detaches a background process keeps the pipes open] → The 2 s reader join makes the call return anyway; the background process is left running (documented non-goal).
- [Killing by `taskkill /T` misses descendants whose parent already exited] → Typical commands (`python`, `pytest`, `npm`, `git`) keep their tree; a job object is the escape hatch if this shows up in practice.
- [Polling every 50 ms wastes a little CPU] → Negligible next to a spawned shell; the alternative breaks Ctrl+C on Windows.
- [A timeout is an error result whose partial output may be useful to the user but is hidden by the summary] → The full text is sent to the model, which normally restates what matters; `scripts/run_tool.py` prints the whole output.
- [`$?`-based suffix returns exit 1 for a failing cmdlet even when an earlier native command had set another `$LASTEXITCODE`] → In that mixed case the code is the stale native one or 1; both are non-zero and the stderr text explains the failure.
- [Tests spawn real shells and are slower on Windows] → About 15 process tests of 0.3-1.5 s; the timeout tests use 1-2 s limits; everything else (blocklist, capture, argv) is process-free.
- [PowerShell 5.1 vs 7 syntax differences (`&&`, `?:`) confuse the model] → The description names the shell in use.
- [Unbounded `pip install`-style commands hit the 600 s ceiling] → The model can split the work; a larger ceiling would block the REPL for too long without streaming.

## Migration Plan

No data, config or CLI migration; `pip install -e ".[dev]"` is unchanged. The tool is additive: the registry gains a seventh tool, existing tools and the `ToolResult` contract are untouched, and the only shared code change is the error branch of `ToolRenderer.show_result`, which behaves as before for results without a summary. Rollback: revert the PR; the agent then simply no longer sees a `shell` tool.

## Open Questions

- Should the terminal show a different indicator (for example yellow) when a successful `shell` result has a non-zero exit code? Needs a `ToolResult` field or a convention on `summary`; deferred, see "Decisions needed" in the change summary.
- Should RRG-69's `auto-edit` level treat `shell` as always requiring approval? Not decided here; the flag is simply `True`.
