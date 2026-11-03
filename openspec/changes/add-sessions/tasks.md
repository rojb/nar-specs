# Tasks

> Review workload: about 450 lines of code and docs plus 550 lines of tests, above the ~400-line guide (a planning heuristic, not a cap; tests are never dropped to fit). Split into two stacked PRs: PR A = group 2, PR B = groups 3 and 4. Retarget the child to `main` before merging the parent with branch deletion.

## [specs]

- [x] 0.1 Link this change to Linear RRG-76 (move the issue to In Progress and add the `nar-specs` change path as a link) and verify the link opens `openspec/changes/add-sessions/` on GitHub once merged
- [ ] 0.2 Open a PR in `nar-specs` from Linear's RRG-76 branch with proposal, specs, design and tasks; verify `openspec validate add-sessions --strict` passes

## [nar]

## 1. Branch

- [x] 1.1 In `nar/`, update `main` and create `nathaliascode/rrg-76-1-store`; verify a clean tree and record which tests already fail _Evidence: on `main`, 958 passed and `test_keyboard_interrupt_kills_the_tree_and_propagates` fails on Linux, already accepted by Nathalia._

## 2. Store and recorder (PR A)

- [ ] 2.1 Create `src/nar/session/store.py` (design.md §2) with `tests/test_session_store.py`; verify a round trip keeps tool calls, errors and non-ASCII text, a failed write leaves the previous file, the file mode is 0600 on Linux, an invalid file or version raises naming the file, an id with `/` or `..` is rejected, `find` handles exact, unique, unknown and ambiguous beginnings, `list` sorts by update date and counts invalid files, and `latest` honors the folder and the exclusion
- [ ] 2.2 Create `src/nar/session/recorder.py` (design.md §3) with `tests/test_session_recorder.py`; verify save writes the context's messages, no messages deletes the file and writes nothing new, a failing save warns once and warns again only after a success, and `switch` moves later saves to the new session

## 3. CLI (PR B)

- [ ] 3.1 Add `recorder` to `run_request` and create it in single and interactive mode, extending `tests/test_cli.py`; verify the scenarios of "Saved sessions": saved after a request, a session interrupted during a tool has every call answered, a cancelled request leaves no trace, compacted messages are saved, and `exit` creates nothing
- [ ] 3.2 Add `--resume` (design.md §4), extending `tests/test_cli.py`; verify every scenario of "Resuming from the command line", that a missing session fails without a provider key, and that `--help` does not import `nar.session`
- [ ] 3.3 Create `src/nar/ui/session_view.py` (design.md §6) with `tests/test_session_view.py`; verify every scenario of "Earlier history is shown"
- [ ] 3.4 Register `/sessions` and `/resume`, extending `tests/test_slash.py`; verify every scenario of "Session commands"

## 4. Docs and checks (PR B)

- [ ] 4.1 Document sessions in the README, with where the files are, that they hold plain text, and that approvals are asked again
- [ ] 4.2 Run `ruff check .`, `ruff format --check .` and `pytest`; verify the only failure, if any, is the one recorded in 1.1, then push and open the stacked PRs referencing RRG-76

## 5. Manual verification

- [ ] 5.1 Walk through RRG-76's "done when" ("close the terminal, run `nar --resume` and continue the conversation where it stopped") at the `nar/` repo root with `NAR_HOME` pointing at a temp folder: with OpenRouter, give `nar` a code name and have it read a file, close the process with `exit`, run `nar --resume`, record the history shown and ask for the code name; kill a run with SIGKILL during a request and confirm it resumes; run `/sessions` and `/resume <id>`; run `nar --resume <id> "prompt"` in single mode. Ollama is not run, at Nathalia's request (2026-11-03). Anthropic is pending until the team has an `ANTHROPIC_API_KEY` (decision of 2026-10-27)
