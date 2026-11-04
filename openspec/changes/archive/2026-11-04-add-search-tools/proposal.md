# Proposal

- **Owner:** nathaliascode
- **Linear issue:** RRG-66
- **Video timestamps covered:** 10:06:07 (Grep vs Glob), 10:08:27 (Grep Tool), 10:37:19 (Glob Tool)
- **Depends on:** RRG-62 `add-tool-framework` (archived)
- **Unblocks:** nothing directly; RRG-73 `add-subagents` uses these tools for its read-only `explorer` subagent
- **Course requirement covered:** none directly

## Why

With `read_file` and `list_dir`, the agent can only find code by listing folders one at a time and reading whole files, which is slow and fills the context. Finding where something is defined or which files exist is the most common first step of any coding task, so the agent needs a way to search file names and file contents across the project, skipping what the project itself ignores.

## What Changes

- Add the built-in **`glob`** tool: finds files whose path matches a pattern such as `**/*.py` or `src/**/test_*.py`, under the working directory or a given folder, and returns their paths, with a cap on the number of results.
- Add the built-in **`grep`** tool: searches file contents with a regular expression and returns matches as `path:line:text`, optionally limited to files matching a name pattern and case-insensitive, with a cap on the number of matches and on the length of each shown line.
- Both tools **skip files ignored by git** (`.gitignore` and the other git exclude files) when the folder is inside a git repository, always skip the noise folders `list_dir` already hides (`.git`, `node_modules`, `.venv`, `__pycache__`, …), and `grep` skips binary and very large files.
- Neither tool requires approval (they only read).
- Register both as built-ins, so the agent (RRG-63) can use them and `scripts/run_tool.py` can run them.
- Add one line to the agent's system prompt suggesting `glob`/`grep` to locate code before reading files.

Out of scope: replacing text across files (that is `edit_file`'s job), multiline regex matches, searching inside archives, a `ripgrep` dependency.

**"Done when" (RRG-66):** the agent finds where a function is defined in a real repository (for example, `load_config` in the `nar` repo).

## Capabilities

### New Capabilities

- `glob-tool`: behavior of the built-in `glob` tool, including pattern rules, ignored files and the result cap.
- `grep-tool`: behavior of the built-in `grep` tool, including regex errors, ignored, binary and large files, and the match cap.

### Modified Capabilities

<!-- None: the tool contract, the registry export and the agent's behavior do not change at spec level. The system prompt line is wording, not a requirement. -->

## Impact

- **Code (`nar` repo):** new `src/nar/tools/search.py` (shared file walker that honors git ignores, pattern matching, path display), `src/nar/tools/glob.py`, `src/nar/tools/grep.py`; registration in `src/nar/tools/builtin.py`; one line in `src/nar/agent/prompt.py`; README section.
- **Dependencies:** none new. Git ignores are applied by asking `git` (`git ls-files`) when it is installed and the folder is a repository; otherwise only the noise folders are skipped.
- **Interfaces other changes rely on:** RRG-73's `explorer` subagent gets `glob` and `grep` in its read-only tool subset.
- **User-visible:** the agent can now answer "where is X defined?" by searching instead of reading every file.
