# Proposal

- **Owner:** Ruben
- **Linear issue:** RRG-83
- **Video timestamps covered:** 17:39:50 (slash commands in the TUI) — the same material as RRG-78; this change corrects the output stream its global rule introduced
- **Depends on:** RRG-78 `add-tui-commands`, RRG-74 `add-mcp-client`, RRG-76 `add-sessions`, RRG-77 `add-checkpoints` (all merged and archived)
- **Unblocks:** nothing; it restores a promise the `tui-commands` spec already makes
- **Course requirement covered:** none of the listed ones; it is a correctness fix on the command surface

## Why

The archived `tui-commands` spec states a global rule: *No command SHALL write its output to standard
output: it goes to standard error as literal text, so a redirected run keeps only the answer.* The
real-model smoke test of 2026-11-05 (`nar` at `4ee96b8`) showed that three commands break it.

`/mcp`, `/sessions` and `/checkpoints` print their answer to standard output, because the changes that
introduced them chose that stream on purpose ("it is the command's answer") before the global rule
existed. Their designs say so explicitly, so this is a contradiction between two specs, not a typo: a
run such as `printf '/sessions\n' | nar > out.txt` puts session ids and model names into a file the
README promises will hold only the answer. The tests build two consoles and never assert which one the
command used, so the suite cannot see it.

## What Changes

- **The three commands answer on standard error.** `/mcp`, `/sessions` and `/checkpoints` pass the
  error console to their views, like every other command already does.
- **Tests lock the stream.** The unit and CLI tests that asserted standard output move to standard
  error and also assert that standard output stayed empty.
- **The spec names the commands.** `tui-commands` keeps its rule and gains scenarios that name the
  three commands and the redirected run, so the contradiction cannot silently return.

Out of scope: the view functions' signatures (`render_mcp_status`, `render_sessions`,
`render_checkpoints` keep taking the console to print to), the banner and the prompt, which belong on
standard output, and the archived designs, which stay as the historical record of what those changes
decided.

**"Done when" (RRG-83):** `/mcp`, `/sessions` and `/checkpoints` answer on standard error, standard
output holds only the banner and the prompt, and `pytest` plus `ruff` pass.

## Capabilities

### New Capabilities

None.

### Modified Capabilities

- `tui-commands`: "The base commands" keeps its rule and gains two scenarios — the commands other
  changes register answer on standard error, and a redirected interactive run keeps only the answers.

## Impact

- **Code (`nar` repo):** `src/nar/cli/slash.py` (the three handlers),
  `tests/test_slash.py` (six tests) and `tests/test_cli.py` (three tests).
- **Behavior:** the three commands move from standard output to standard error. Anyone who parsed
  `/mcp` or `/sessions` output from standard output has to read standard error, which is what the
  README already documents.
- **Dependencies:** none.
- **Specs:** one MODIFIED requirement in `tui-commands`; no ordering constraint, since that capability
  already exists in the store.
