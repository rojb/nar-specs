# Tasks

> Review workload: about 300 lines of code and docs plus 350 lines of tests, near the ~400-line guide (a planning heuristic, not a cap; tests are never dropped to fit). No stacked PRs are needed: it fits in one PR.

## [specs]

- [x] 0.1 Link this change to Linear RRG-77 (move the issue to In Progress and add the `nar-specs` change path as a link) and verify the link opens `openspec/changes/add-checkpoints/` on GitHub once merged
- [x] 0.2 Open a PR in `nar-specs` from Linear's RRG-77 branch with proposal, specs, design and tasks; verify `openspec validate add-checkpoints --strict` passes

## [nar]

## 1. Branch

- [x] 1.1 In `nar/`, update `main` and create `nathaliascode/rrg-77-add-checkpoints-guardar-y-restaurar-puntos-de-control`; verify a clean tree and record which tests already fail _Evidence: on `main`, 1060 passed and `test_keyboard_interrupt_kills_the_tree_and_propagates` fails on Linux, already accepted by Nathalia._

## 2. Store

- [x] 2.1 Create `src/nar/session/checkpoints.py` (design.md §2 and §3) with `tests/test_session_checkpoints.py`; verify a round trip keeps tool calls, numbers are never reused, names are validated, a corrupt file raises naming it and is not overwritten, the file mode is 0600 on Linux, and `SessionStore.list` shows no extra entry and counts no invalid file

## 3. Recorder and commands

- [x] 3.1 Add `checkpoint`, `checkpoints` and `restore` to `SessionRecorder` (design.md §4), extending `tests/test_session_recorder.py`; verify a restore discards later messages in the context and in the session file, an unknown reference changes nothing, later checkpoints remain, and they follow the session after `switch`
- [x] 3.2 Add `render_checkpoints` and register the three commands (design.md §5), extending `tests/test_session_view.py`, `tests/test_slash.py` and `tests/test_cli.py`; verify every scenario of `specs/checkpoints/spec.md`, including the request after a restore sent with the checkpoint's messages only, and a checkpoint available after `nar --resume`

## 4. Docs and checks

- [x] 4.1 Document the commands and the difference between a session and a checkpoint in the README, and that files are not restored
- [x] 4.2 Run `ruff check .`, `ruff format --check .` and `pytest`; verify the only failure, if any, is the one recorded in 1.1, then push and open the PR referencing RRG-77

## 5. Manual verification

- [ ] 5.1 Walk through RRG-77's "done when" ("restoring a checkpoint discards the later messages and the agent continues from there") at the `nar/` repo root with `NAR_HOME` pointing at a temp folder: with OpenRouter, give `nar` a code name, run `/checkpoint base`, give a second code name, run `/restore base`, ask for the second one and record that the model does not know it, and that it still knows the first; open the session file and record that the discarded request is gone; leave, run `nar --resume`, run `/checkpoints` and restore again. Ollama is not run, at Nathalia's request (2026-11-03). Anthropic is pending until the team has an `ANTHROPIC_API_KEY` (decision of 2026-10-27)

  _Evidence (OpenRouter, temporary `NAR_HOME`, 2026-11-04): gave the code name GATO-42, ran `/checkpoint base` (`checkpoint 1 saved: base (2 messages)`), gave a second code name LORO-7, and `/checkpoints` listed the checkpoint. `/restore base` said the conversation had 2 messages and files were not restored. Asked for the second code name, the model answered "NO LO SE" and still gave GATO-42. The session file no longer contained LORO-7. After `exit` and `nar --resume`, `/checkpoints` listed it, `/restore base` worked, and `/sessions` showed one session with no invalid-file warning._

  _Not run with Ollama, at Nathalia's request (2026-11-03). Still open: Anthropic._
