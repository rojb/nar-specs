# Proposal

- **Owner:** Ruben
- **Linear issue:** RRG-65
- **Video timestamps covered:** 08:58:15 (Shell Tool)
- **Depends on:** RRG-62 `add-tool-framework` (archived). Also builds on the merged agent loop (RRG-63), whose approval gate decides when `requires_approval` tools run.
- **Unblocks:** RRG-69 `add-approval-system` (needs a tool whose approval prompt shows the command to run; its roadmap row waits for RRG-64 and RRG-65), RRG-72 `add-hooks` (`before_tool` can block shell commands).
- **Course requirement covered:** none directly; `shell` is the riskiest tool the agent has, so it is the main case for human in the loop (RRG-69), which reads its `requires_approval` flag.

## Why

With the file tools the agent can read and change a project, but it cannot run anything: it cannot check the Python version, run the tests, start a linter or call `git`. Running commands is what turns a code assistant into an agent that can verify its own work. It is also the most dangerous capability nar has: a command can delete files, hang forever or produce megabytes of output. The tool must therefore bound time and output, work on the user's real shell (PowerShell on Windows, bash elsewhere), refuse the most obviously destructive commands and be marked as requiring approval.

## What Changes

- Add the built-in **`shell`** tool: runs one command line in the platform's shell, with the working directory where `nar` was started, and returns the exit code, stdout and stderr. Marked `requires_approval`.
  - **Shell:** PowerShell on Windows (`pwsh` when installed, otherwise Windows PowerShell), `bash` on Linux/macOS (`sh` when bash is missing). No shell profile is loaded and standard input is closed, so a command that waits for input ends instead of hanging.
  - **Timeout:** an optional `timeout` argument in seconds (default 120, at most 600). When it expires the whole process tree is killed and the result reports the timeout together with the output captured so far.
  - **Output:** stdout and stderr are reported separately, the exit code is always reported, and each stream is limited to 14,000 characters keeping its beginning and its end (the end is where build and test failures are). Memory use is bounded even for commands that print endlessly.
  - **A non-zero exit code is not a tool error**: the command ran and its result is information for the model. Only things that stop the command from completing (blocked, timed out, could not start) are error results.
  - **Blocklist:** commands matching a small list of obviously destructive patterns (`rm -rf /`, `rm -rf ~`, `mkfs`, `dd of=/dev/...`, a fork bomb, `shutdown`, `format C:`, `Remove-Item -Recurse -Force C:\`, `del /s /q C:\` and similar) are refused and never run. This is a safety net against model mistakes, not a sandbox.
  - **Ctrl+C** while a command runs kills the command and its children before the request is cancelled.
- Let the terminal show a short summary for **error results that carry one** (`timed out after 5 s · 2 lines`) instead of dumping the whole message, so a timed-out command that printed a lot does not flood the screen. The model still receives the full text.
- Register `shell` as a built-in, so it appears in the provider-neutral export, the agent can call it and `scripts/run_tool.py` can run it.

Out of scope: asking the user before a command runs (RRG-69 reads the flag; until then the agent loop runs `shell` only with `approval = "yolo"`), sandboxing, background or long-running processes, interactive commands and pseudo-terminals, per-command allow-lists, a configurable shell or timeout, streaming output while the command runs, and changing the `cwd` between calls.

**"Done when" (RRG-65):** the agent runs `python --version` and uses the output in its answer.

## Capabilities

### New Capabilities

- `shell-tool`: behavior of the built-in `shell` tool: shell selection, result report, timeout, output limit, blocklist, cancellation and registration.

### Modified Capabilities

- `tool-framework`: the tool activity display requirement gains showing a result's summary instead of the full message when an error result carries one.

## Impact

- **Code (`nar` repo):** new `src/nar/tools/shell.py` (the tool), `shell_process.py` (shell selection, bounded capture, running and killing the process tree) and `shell_blocklist.py` (the dangerous-command patterns); built-in registration in `src/nar/tools/builtin.py`; one small change in `src/nar/ui/tool_view.py`; README section for the tool.
- **Dependencies:** none (`subprocess`, `threading`, `shutil`, `re` are standard library).
- **Configuration:** no new keys in `~/.nar/config.toml` or `.nar/config.toml`.
- **Interfaces other changes rely on:** RRG-69 reads `requires_approval` and shows `arguments["command"]` in its prompt; RRG-72 hooks can inspect the same argument; RRG-63's approval gate already refuses the tool below `yolo`.
- **User-visible:** no new `nar` command or flag; the tool is visible through `scripts/run_tool.py` now and, with `approval = "yolo"`, through the agent.
