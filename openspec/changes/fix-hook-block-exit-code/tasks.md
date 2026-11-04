# Tasks

> Review workload: about 30 lines of code plus 60 of tests and docs, well under the ~400-line guide.
> One PR. The branch name and the "done when" are the ones the Linear issue shows, read on 2026-11-04.

## [specs]

- [ ] 0.1 Link this change to Linear RRG-82 (move the issue to In Progress and add the `nar-specs` change path as a link on the issue); verify the link opens `openspec/changes/fix-hook-block-exit-code/` on GitHub once merged
- [ ] 0.2 Open a PR in `nar-specs` from Linear's RRG-82 branch with proposal, specs, design and tasks; verify `openspec validate fix-hook-block-exit-code --strict` passes and that `openspec validate hooks --strict` rejects nothing after the archive

## [nar]

## 1. Branch

- [ ] 1.1 In `nar/`, update `main` and create `rubenjustiniano96/rrg-82-fix-hook-block-exit-code-un-hook-roto-no-debe-bloquear-la` (the `gitBranchName` of RRG-82); verify a clean tree and that `pytest -q` and `ruff check .` pass before any change _Evidence: baseline on `main` at `7f360bc`: 1240 passed, 3 skipped; ruff clean; 143 files formatted._

## 2. The block code

- [ ] 2.1 In `src/nar/hooks/runner.py` add `BLOCK_EXIT_CODE = 2` with the reason in a comment, make `before_tool` block only on that code (not on a startup error or a timeout), and split the warning of a non-zero exit into "refused the call" and "failed with exit code N and did not block"; extend `tests/test_hooks_runner.py`; verify `pytest tests/test_hooks_runner.py` passes for: exit 2 blocks with the hook's text and `FailureKind.REFUSED`, exit 0 lets the call run, exit 1 warns `did not block` and returns `None`, exit 3 behaves like exit 1, a startup error and a timeout still do not block and now say so, and — the regression this change exists for — a `before_tool` hook whose command does not exist does not block, using the real shell from `find_shell()` (skipped when the machine has none)
- [ ] 2.2 Update `README.md`: the block code is `2`, any other non-zero code means the hook failed and the call runs, and the `before_tool` example uses `exit 2`; verify every command in the section runs as written and that no sentence still says a non-zero exit blocks

## 3. Checks and delivery

- [ ] 3.1 Run `ruff check .`, `ruff format --check .` and `pytest`; verify all three pass with only the pre-existing skips _Evidence: <commit>_
- [ ] 3.2 Push and open a PR in `nar` referencing RRG-82 with a Conventional Commit message (for example `fix(hooks): block only on exit code 2 (RRG-82)`); verify the PR shows only this change, then merge it and confirm `main` is clean
- [ ] 3.3 Archive the change in `nar-specs`, confirm `openspec validate --all --strict` is fully green, and link the archive on RRG-82

## 4. Manual verification (RRG-82 "done when")

- [ ] 4.1 In a real terminal with `deepseek`, `NAR_HOME` in a temp folder and a temp project, record what was observed for each step:
  - Broken hook: `before_tool = ["this-command-does-not-exist"]`, ask for an edit; record the warning with `did not block` and that the edit happened
  - Refusing hook: `before_tool = ["Write-Output 'blocked by policy'; exit 2"]`, ask for an edit; record that the file stayed untouched, the refusal line, and the model reacting to the hook's sentence
  - One-day-old mistake: `exit 1` in place of `exit 2`; record that the edit ran and that the warning says the call was not blocked
  - Recording: date, shell, provider and model, `nar` commit, and the exact warning lines
  - _Evidence: <date, environment, observations>_
