# Proposal

- **Owner:** nathaliascode
- **Linear issue:** RRG-75
- **Video timestamps covered:** 13:02:55 (Tool Discovery: Build Your Own Tool)
- **Depends on:** RRG-62 `add-tool-framework` (archived; `Tool`, `ToolRegistry`)
- **Unblocks:** nothing
- **Course requirement covered:** none directly

## Why

Adding a tool today means changing `nar`'s code. A user should be able to add one by dropping a Python file in a folder.

Loading that file runs it. A file in `.nar/tools/` of a cloned repository comes from whoever wrote the repository, so opening `nar` there must not run it without asking. Nathalia decided on 2026-11-04 that project tools ask for confirmation once per project, and that global tools load without asking, because the user put them there.

## What Changes

- **Discovery at startup:** `nar` loads the `.py` files of `~/.nar/tools/` (global, `NAR_HOME` respected) and `.nar/tools/` (project) that define subclasses of `Tool`, and registers them in the `ToolRegistry`. Files starting with `_` are skipped.
- **Confirmation for project tools:** before running anything from `.nar/tools/`, `nar` lists the files and asks. The answer is remembered for that folder and those exact contents, so a changed or new file asks again.
- **When `nar` cannot ask** (a pipe, CI), project tools are not loaded, with a warning. `--trust-project-tools` allows them explicitly. A setting in a file cannot allow them, and `--approval yolo` does not either, because a repository can ship its own `.nar/config.toml`.
- **A file that fails never breaks `nar`:** one warning naming the file, and the other tools still load. This covers import errors, `sys.exit`, a tool that cannot be created, and a repeated name.
- **Built-in tools cannot be replaced:** a repeated name is a warning and the first one stays, in the order built-in, global, project.
- **Approval:** a custom tool follows its own `requires_approval`, as built-in tools do.
- **Example tool and short guide** in the README of `nar`.

Out of scope: a sandbox for custom tools, installing their dependencies, reloading during a session, listing or revoking trusted folders from a command (the file can be edited), trusting a project through configuration.

**"Done when" (RRG-75):** dropping a `.py` in `.nar/tools/` lets the agent use that tool without touching `nar`'s code. For a project tool, this holds after the user confirms once.

## Capabilities

### New Capabilities

- `custom-tools`: finding and loading tool files, confirmation for project tools, failures, name collisions, and the guide.

### Modified Capabilities

None.

## Impact

- **Code (`nar` repo):** new `src/nar/tools/custom.py`; `src/nar/cli/runner.py` (`create_agent` loads them), `src/nar/cli/main.py` and `src/nar/cli/repl.py` (`--trust-project-tools`), new `src/nar/ui/trust_prompt.py`; new `examples/tools/word_count.py`; README.
- **User filesystem:** reads `~/.nar/tools/` and `.nar/tools/`; writes `~/.nar/trusted_tools.json`.
- **Security:** a custom tool is ordinary Python with the user's permissions; nothing limits it once it is loaded. The confirmation decides whether it runs at all.
- **Dependencies:** none new.
- **Specs:** only a new capability.
