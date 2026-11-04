# Tasks

> Review workload: about 30 lines of code plus 60 of tests and docs, well under the ~400-line guide.
> One PR. The branch name and the "done when" are the ones the Linear issue shows, read on 2026-11-04.

## [specs]

- [x] 0.1 Link this change to Linear RRG-82 (move the issue to In Progress and add the `nar-specs` change path as a link on the issue); verify the link opens `openspec/changes/fix-hook-block-exit-code/` on GitHub once merged _Evidence: RRG-82 created on 2026-11-04 with the change path attached; the issue states the problem, the scope and why the block code is 2 rather than 1._
- [x] 0.2 Open a PR in `nar-specs` from Linear's RRG-82 branch with proposal, specs, design and tasks; verify `openspec validate fix-hook-block-exit-code --strict` passes and that `openspec validate hooks --strict` rejects nothing after the archive _Evidence: PR rojb/nar-specs#19; `openspec validate fix-hook-block-exit-code --strict` valid and `validate --all --strict` 29 passed, 0 failed before the archive._

## [nar]

## 1. Branch

- [x] 1.1 In `nar/`, update `main` and create `rubenjustiniano96/rrg-82-fix-hook-block-exit-code-un-hook-roto-no-debe-bloquear-la` (the `gitBranchName` of RRG-82); verify a clean tree and that `pytest -q` and `ruff check .` pass before any change _Evidence: baseline on `main` at `4ee96b8`'s parent, `7f360bc`: 1240 passed, 3 skipped; ruff clean; 143 files formatted._

## 2. The block code

- [x] 2.1 In `src/nar/hooks/runner.py` add `BLOCK_EXIT_CODE = 2` with the reason in a comment, make `before_tool` block only on that code (not on a startup error or a timeout), and split the warning of a non-zero exit into "refused the call" and "failed with exit code N and did not block"; extend `tests/test_hooks_runner.py`; verify `pytest tests/test_hooks_runner.py` passes for: exit 2 blocks with the hook's text and `FailureKind.REFUSED`, exit 0 lets the call run, exit 1 warns `did not block` and returns `None`, exit 3 behaves like exit 1, a startup error and a timeout still do not block and now say so, and — the regression this change exists for — a `before_tool` hook whose command does not exist does not block, using the real shell from `find_shell()` (skipped when the machine has none) _Evidence: nar `fceddfd`; `pytest tests/test_hooks_runner.py` 15 passed, including `test_a_missing_command_does_not_block_with_the_real_shell`, which runs the actual Windows PowerShell and asserts the call is not blocked. Deviation: the old tests blocked with `exit 1` and were rewritten to the new code, and `tests/test_subagent.py` (written by nathaliascode for RRG-73) blocked the same way, so its `sys.exit(1)` became `sys.exit(2)` with a comment; her production code never depended on a specific code, only that test did._
- [x] 2.2 Update `README.md`: the block code is `2`, any other non-zero code means the hook failed and the call runs, and the `before_tool` example uses `exit 2`; verify every command in the section runs as written and that no sentence still says a non-zero exit blocks _Evidence: nar `fceddfd`; the `Hooks` section now states that `before_tool` refuses with code `2`, that any other non-zero code (including `1`, which the shell returns for a missing command) means the hook failed and the call runs, and the broken-hook bullet says a mistyped hook never blocks. No command was added to the example, so nothing new to run._

## 3. Checks and delivery

- [x] 3.1 Run `ruff check .`, `ruff format --check .` and `pytest`; verify all three pass with only the pre-existing skips _Evidence: nar `fceddfd`: `ruff check .` All checks passed; `ruff format --check .` 143 files already formatted; `pytest -q` 1242 passed, 3 skipped in 2m 01s (baseline 1240, so +2 tests)._
- [x] 3.2 Push and open a PR in `nar` referencing RRG-82 with a Conventional Commit message (for example `fix(hooks): block only on exit code 2 (RRG-82)`); verify the PR shows only this change, then merge it and confirm `main` is clean _Evidence: PR rojb/nar#35, one commit `fceddfd` (95 lines), merged as `4ee96b8` with the branch deleted; `main` carries `BLOCK_EXIT_CODE = 2`._
- [x] 3.3 Archive the change in `nar-specs`, confirm `openspec validate --all --strict` is fully green, and link the archive on RRG-82 _Evidence: archived as `2026-11-04-fix-hook-block-exit-code`; `openspec validate --all --strict` fully green afterwards._

## 4. Manual verification (RRG-82 "done when")

- [x] 4.1 In a real terminal with `deepseek`, `NAR_HOME` in a temp folder and a temp project, record what was observed for each step:
  - Broken hook: `before_tool = ["this-command-does-not-exist"]`, ask for an edit; record the warning with `did not block` and that the edit happened
  - Refusing hook: `before_tool = ["Write-Output 'blocked by policy'; exit 2"]`, ask for an edit; record that the file stayed untouched, the refusal line, and the model reacting to the hook's sentence
  - One-day-old mistake: `exit 1` in place of `exit 2`; record that the edit ran and that the warning says the call was not blocked
  - Recording: date, shell, provider and model, `nar` commit, and the exact warning lines
  - _Evidence: VERIFIED 2026-11-04 on Windows 11 with Windows PowerShell, `deepseek-flash`, `nar` at `4ee96b8`, temp `NAR_HOME` and a temp project. A) the hook `this-command-does-not-exist` produced `warning: hook before_tool failed with exit code 1 and did not block the call: this-command-does-not-exist` twice and the edit landed (`# demo.py` appeared), so the bug is gone. B) `Write-Output "blocked by policy"; exit 2` produced `error the before_tool hook refused this call: blocked by policy` and the file stayed untouched. C) `Write-Output "oops"; exit 1` produced the `did not block the call` warning and the edit landed. Before this change, A and C blocked every call in silence._
