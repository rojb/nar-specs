# Proposal

- **Owner:** Ruben
- **Linear issue:** RRG-64
- **Video timestamps covered:** 07:44:21 (Write File Tool), 08:25:08 (Edit File Tool), 09:45:51 (List Dir Tool)
- **Depends on:** RRG-62 `add-tool-framework` (archived)
- **Unblocks:** RRG-69 `add-approval-system` (needs tools with `requires_approval` and a diff to show), RRG-72 `add-hooks` (its example runs ruff after `edit_file`)
- **Course requirement covered:** none directly; `write_file` and `edit_file` are the first tools that require approval, which human in the loop (RRG-69) builds on.

## Why

With only `read_file`, the agent can look at a project but cannot change it. Creating files, editing them precisely and seeing what is in a folder are the minimum actions a coding agent needs, and they are the first tools whose side effects the user must be able to see (a diff) and later approve (RRG-69). The tool framework from RRG-62 is merged, so these tools can be added now through the registry without waiting for the agent loop.

## What Changes

- Add the built-in **`write_file`** tool: creates a file or overwrites an existing one with the given content, creating missing parent folders, and reports what it did (created or overwritten, size). Marked `requires_approval`.
- Add the built-in **`edit_file`** tool: replaces an exact `old_string` with `new_string` in a file. It fails with a clear error when `old_string` is not found or appears more than once, so the model must give enough context for a unique match. On success it reports a diff of the change. Marked `requires_approval`.
- Add the built-in **`list_dir`** tool: lists the entries of a folder (files and subfolders distinguished), skipping noise folders such as `.git`, `node_modules`, `.venv` and `__pycache__`. Does not require approval.
- Show the **diff** of file changes in the terminal when a tool result carries one, with added and removed lines visually distinct, extending how tool activity is displayed.
- Register the three tools as built-ins, so they appear in the provider-neutral export and can be run with `scripts/run_tool.py`.

Out of scope: asking the user before running `write_file`/`edit_file` (RRG-69 reads the flag), the shell tool (RRG-65), search tools (RRG-66), confining paths to the project folder (an approval/autonomy concern for RRG-69).

**"Done when" (RRG-64):** the agent creates a file, edits it, and the diff is shown in the terminal. The agent loop (RRG-63, nathaliascode) is not merged yet, so this change verifies the same flow by running the tools through `scripts/run_tool.py`; the end-to-end check with the agent is repeated once RRG-63 lands.

## Capabilities

### New Capabilities

- `write-file-tool`: behavior of the built-in `write_file` tool.
- `edit-file-tool`: behavior of the built-in `edit_file` tool, including the unique-match rule and the reported diff.
- `list-dir-tool`: behavior of the built-in `list_dir` tool, including ignored folders.

### Modified Capabilities

- `tool-framework`: the tool activity display requirement gains showing a diff for tool results that change files.

## Impact

- **Code (`nar` repo):** new `src/nar/tools/write_file.py`, `edit_file.py`, `list_dir.py`; built-in registration in `src/nar/tools/builtin.py`; diff rendering in `src/nar/ui/tool_view.py`; possibly a diff field on `ToolResult` in `src/nar/tools/base.py` (decided in design); README section for the new tools.
- **Dependencies:** none expected (`difflib` is in the standard library).
- **Interfaces other changes rely on:** RRG-69 reads `requires_approval` and shows the diff before approval; RRG-72 hooks react to `edit_file`; RRG-63 exposes the tools to the model through the registry export.
- **User-visible:** no new `nar` command or flag; the tools are visible through `scripts/run_tool.py` now and through the agent once RRG-63 is merged.
