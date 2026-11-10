# Tasks

> Review workload: about 350 lines, mostly tests. It fits in one PR.

## [specs]

- [x] 0.1 Link this change to Linear RRG-87 and verify the link opens `openspec/changes/add-api-key-prompt/` on GitHub once merged
- [x] 0.2 Open a PR in `nar-specs` from Linear's RRG-87 branch; verify `openspec validate add-api-key-prompt --strict` passes _Evidence: rojb/nar-specs#22; the validation passed._

## [nar]

- [x] 1.1 Create Linear's RRG-87 branch from `main`, and add `KeyPrompter` in `src/nar/ui/key_prompt.py`; verify with injected readers: the hidden key read, an empty answer, Ctrl+C, end of input, and the save question _Evidence: `tests/test_api_key.py`._
- [x] 1.2 Add `obtain_api_key`, `PendingKey`, `keep_api_key`, `prepare_api_key` and `save_user_variable` in `src/nar/cli/api_key.py`; verify the following:
  - nothing is asked without a terminal, with a key already set, or for Ollama
  - the variable is set for the process, and `discard` removes it
  - saving on Windows calls the registry writer
  - other systems print the `export` line without the key

  _Evidence: `tests/test_api_key.py`. `save_user_variable` was also run against the real registry. A new PowerShell process read the test variable, which was then removed._
- [x] 1.3 Call it at startup (in `create_agent`, see design) and in `/model`; verify with fake providers in `tests/test_cli.py`:
  - a valid key starts the session
  - a rejected key exits with code 1 without saving
  - `/model` keeps the previous provider on rejection
  - the key never appears in standard output or standard error
  - piped runs behave as before
  - `ruff check .`, `ruff format --check .` and `pytest` are clean

  _Evidence: rojb/nar#38. `ruff` was clean, and `pytest` gave 1286 passed and 3 skipped. An autouse fixture makes any real save fail a test._

## Manual verification

- [x] 2.1 Walk through RRG-87's "done when" in a real Windows terminal:
  - remove `DEEPSEEK_API_KEY` from the user environment and open a new terminal
  - run `nar --provider deepseek`, paste the key and record that it is hidden
  - answer `y`, open another new terminal, and record that `nar --provider deepseek` starts without asking
  - record that a wrong key is rejected and not saved

  _Evidence (Windows 11, 2026-11-10, `pruebas/demo-rrg86-87.py`, 18 of 18 checks, run while Nathalia watched): `nar` ran in a real Windows Terminal console, and the keys were typed into it with `WriteConsoleInput`, so `getpass` read them as it reads a person's typing. The team has no real key, so the walk-through used `anthropic` against a simulated Anthropic server reached with `ANTHROPIC_BASE_URL`, instead of DeepSeek. The demo recorded the following:_
  - _A piped run asked nothing and exited with code 1 and `ANTHROPIC_API_KEY is not set`._
  - _A wrong key was hidden while typed, was rejected (`anthropic rejected the API key`), exited with code 1, and was not saved in the Windows user environment._
  - _A valid key was hidden while typed and checked. `y` saved it under `HKCU\Environment`, and the model answered with it. The key never appeared on screen._
  - _In the same session, `/model deepseek` asked for `DEEPSEEK_API_KEY`. Enter cancelled it, and the session stayed on `anthropic`._
  - _A new process that took the variable from the Windows user environment started without asking, and got an answer._
  - _The demo then removed `ANTHROPIC_API_KEY` from the user environment._

  _Not run on 2026-11-10 at archive time: the same walk-through with a real DeepSeek key, because the team had none yet._

  _Evidence 2, added after the archive. Nathalia ran it herself on 2026-11-10, in her own PowerShell window with her real DeepSeek key:_
  - _`nar --provider deepseek` printed `DEEPSEEK_API_KEY is not set.` and `Paste your deepseek API key (hidden; Enter to cancel):`. The pasted key was not shown._
  - _Then `checking deepseek · deepseek-flash...` and the save question appeared. `y` printed `saved DEEPSEEK_API_KEY for your Windows user; new terminals will see it`._
  - _DeepSeek answered `hola`._
  - _Later processes read `DEEPSEEK_API_KEY` and `ANTHROPIC_API_KEY` from the Windows user environment and started without asking._
