# Tasks

> Review workload: 3 changed lines of code plus about 50 of tests and docs, well under the ~400-line
> guide. One PR. **Deviation, recorded on purpose:** the fix was written and verified before these
> artifacts existed, because the smoke test of 2026-11-05 found the bug and the owner authorized the
> fix on the spot. Tasks 1.1 and 3.2 stay unchecked: the work still sits uncommitted on `main`, and
> the branch, the commit and the PR are the owner's calls.

## [specs]

- [x] 0.1 Create RRG-83 (title, problem, scope, "done when"), attach it to the `nar` project with the
  `Fase 5 — Sesiones y UX` milestone and the `S`/`Bug` labels, and record its `gitBranchName`
  _Evidence: RRG-83 created on 2026-11-05, In Progress, branch `rubenjustiniano96/rrg-83-fix-slash-command-stream-mcp-sessions-y-checkpoints-escriben`._
- [x] 0.2 Open a PR in `nar-specs` from the RRG-83 branch with proposal, specs, design and tasks;
  verify `openspec validate fix-slash-command-stream --strict` passes and that
  `openspec validate --all --strict` stays green
  _Evidence: commit `2719602` on the RRG-83 branch, PR rojb/nar-specs#20; `openspec validate fix-slash-command-stream --strict` valid and `openspec validate --all --strict` 29 passed, 0 failed._

## [nar]

## 1. Branch

- [x] 1.1 In `nar/`, update `main` and create
  `rubenjustiniano96/rrg-83-fix-slash-command-stream-mcp-sessions-y-checkpoints-escriben` (the
  `gitBranchName` of RRG-83), moving the current working-tree changes there; verify a clean `main`
  and the same `pytest` result on the branch
  _Evidence: branch created from `main` at `4ee96b8`; commit `bce4d68` (3 files, 34 insertions, 26 deletions); `main` is clean. The suite had already run on that exact working tree before committing: 1242 passed, 3 skipped, ruff clean._

## 2. The stream

- [x] 2.1 In `src/nar/cli/slash.py` make `_mcp_command`, `_sessions_command` and
  `_checkpoints_command` pass `context.err` to their views; update `tests/test_slash.py` and
  `tests/test_cli.py` so every assertion about those answers reads standard error and asserts that
  standard output stayed empty; verify the new tests fail before the change and pass after it
  _Evidence: RED observed first — `pytest tests/test_slash.py` failed 7 tests (six stream assertions plus the sessions one) — then GREEN 50 passed; `pytest tests/test_cli.py -k "checkpoint_is_available or interactive_mcp_command"` 3 passed; the earlier full-suite run caught three CLI tests in `tests/test_cli.py` that the unit tests could not see._

## 3. Checks and delivery

- [x] 3.1 Run `ruff check`, `ruff format --check` and `pytest`; verify all three pass with only the
  pre-existing skips
  _Evidence: `ruff check src tests` All checks passed; `ruff format --check src tests` 139 files already formatted; `pytest -q` 1242 passed, 3 skipped in 1m 46s (same count as the baseline before the change, since the work rewrote assertions instead of adding cases)._
- [x] 3.2 Commit the change with a Conventional Commit message referencing RRG-83 (for example
  `fix(cli): answer /mcp, /sessions and /checkpoints on stderr (RRG-83)`), push the branch and open a
  PR in `nar`; verify the PR shows only this change, then merge it and confirm `main` is clean
  _Evidence: commit `bce4d68`, PR rojb/nar#36 with only these three files, merged as `71907f2` (merge commit) with the branch deleted; `main` clean and carrying the fix. Baseline for comparison: the suite on the same tree gave 1242 passed, 3 skipped and ruff clean._
- [x] 3.3 Archive the change in `nar-specs`, confirm `openspec validate --all --strict` is fully green,
  and link the archive on RRG-83
  _Evidence: artifacts merged as `968ca99` (PR rojb/nar-specs#20); change archived as `openspec/changes/archive/2026-11-05-fix-slash-command-stream`, and `openspec validate --all --strict` reports 29 passed, 0 failed afterwards._

## 4. Manual verification (RRG-83 "done when")

- [x] 4.1 In a real terminal, with `deepseek` and a temporary `NAR_HOME`, run an interactive session
  that types `/mcp`, `/sessions` and `/checkpoints` with standard output redirected to a file, and
  record what each stream received, the exit code and the provider/model
  _Evidence: VERIFIED 2026-11-05, Windows 11, Windows PowerShell, `deepseek-flash`, `nar` `4ee96b8`
  plus the working-tree fix, temporary `NAR_HOME` and a temporary project, command
  `printf '/mcp\n/sessions\n/checkpoints\nexit\n' | nar > out.txt 2> err.txt`: exit 0; `out.txt`
  held only `nar · deepseek · deepseek-flash. Type 'exit' to leave.` and the four `nar>` prompts;
  `err.txt` held `No MCP servers are configured. Define one as [mcp_servers.<name>] in .nar/config.toml
  (this project) or ~/.nar/config.toml (all projects).`, `no saved sessions` and
  `this session has no checkpoints`. Before the fix, the same run put all three answers in `out.txt`._
