# Tasks

> Review workload: under 100 lines, mostly tests. It fits in one PR.

## [specs]

- [x] 0.1 Link this change to Linear RRG-91 and verify the link opens `openspec/changes/fix-api-key-paste/` on GitHub once merged
- [x] 0.2 Open a PR in `nar-specs` from Linear's RRG-91 branch; verify `openspec validate fix-api-key-paste --strict` passes _Evidence: rojb/nar-specs#30; the validation passed._

## [nar]

- [ ] 1.1 Create Linear's RRG-91 branch from `main`, and add tests in `tests/test_api_key.py` with injected readers. Verify they fail before the fix:
  - `\x16` alone and after a key is refused with the right-click hint, and the key is asked again
  - quotes and `DEEPSEEK_API_KEY=` are removed
  - inner spaces and other control characters are refused
  - the key never appears in the output
- [ ] 1.2 Add `clean_key` and the loop in `ask_key`. Verify:
  - the tests from 1.1 pass
  - the RRG-87 tests still pass
  - `ruff check .`, `ruff format --check .` and `pytest` are clean

## Manual verification

- [ ] 2.1 Walk through RRG-91's "done when" in a classic console window:
  - with `DEEPSEEK_API_KEY` removed from the process, type Ctrl+V and Enter at the prompt, and record the hint and the new prompt
  - type the real key, and record that DeepSeek accepts it
