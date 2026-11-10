# Tasks

> Review workload: about 350 lines, mostly tests. It fits in one PR.

## [specs]

- [ ] 0.1 Link this change to Linear RRG-87 and verify the link opens `openspec/changes/add-api-key-prompt/` on GitHub once merged
- [ ] 0.2 Open a PR in `nar-specs` from Linear's RRG-87 branch; verify `openspec validate add-api-key-prompt --strict` passes

## [nar]

- [ ] 1.1 Create Linear's RRG-87 branch from `main`, and add `KeyPrompter` in `src/nar/ui/key_prompt.py`; verify with injected readers: the hidden key read, an empty answer, Ctrl+C, end of input, and the save question
- [ ] 1.2 Add `obtain_api_key`, `PendingKey`, `keep_api_key`, `prepare_api_key` and `save_user_variable` in `src/nar/cli/api_key.py`; verify the following:
  - nothing is asked without a terminal, with a key already set, or for Ollama
  - the variable is set for the process, and `discard` removes it
  - saving on Windows calls the registry writer
  - other systems print the `export` line without the key
- [ ] 1.3 Call it at startup and in `/model`; verify with fake providers in `tests/test_cli.py`:
  - a valid key starts the session
  - a rejected key exits with code 1 without saving
  - `/model` keeps the previous provider on rejection
  - the key never appears in standard output or standard error
  - piped runs behave as before
  - `ruff check .`, `ruff format --check .` and `pytest` are clean

## Manual verification

- [ ] 2.1 Walk through RRG-87's "done when" in a real Windows terminal:
  - remove `DEEPSEEK_API_KEY` from the user environment and open a new terminal
  - run `nar --provider deepseek`, paste the key and record that it is hidden
  - answer `y`, open another new terminal, and record that `nar --provider deepseek` starts without asking
  - record that a wrong key is rejected and not saved
