# Tasks

> Review workload: about 350 lines, mostly tests. It fits in one PR.

## [specs]

- [x] 0.1 Link this change to Linear RRG-89 and verify the link opens `openspec/changes/add-provider-fallback/` on GitHub once merged
- [x] 0.2 Open a PR in `nar-specs` from Linear's RRG-89 branch; verify `openspec validate add-provider-fallback --strict` passes _Evidence: rojb/nar-specs#26; the validation passed._

## [nar]

- [x] 1.1 Create Linear's RRG-89 branch from `main`, and add `unavailable` to `ProviderError` and set it in `retry.py`. Verify with mocked HTTP:
  - 402 and Anthropic's "credit balance" 400 are marked
  - retries exhausted on 429 and 529 are marked
  - an unreachable server is marked
  - 401, 404 and other 400s are not marked

  _Evidence: new tests in `tests/test_provider_openai_compat.py` and `tests/test_provider_anthropic.py`. 529 is covered by the same `_exhausted` path as 429, which is tested._
- [x] 1.2 Add `ProviderMenu` in `src/nar/ui/provider_menu.py`. Verify with injected readers: the numbered list and its statuses, a valid pick, Enter, end of input, Ctrl+C, an unknown answer asked again, and no terminal _Evidence: `tests/test_provider_menu.py`._
- [x] 1.3 Add `retry_elsewhere` in `src/nar/cli/fallback.py` and call it from `repl.run`. Verify with fake providers in `tests/test_cli.py`:
  - a pick switches and the same request is answered by the new provider
  - Enter stays
  - a failing check shows the list again
  - a second unavailable failure offers the list without the tried providers
  - unmarked errors, single mode and piped input show no list
  - `ruff check .`, `ruff format --check .` and `pytest` are clean

  _Evidence: rojb/nar#40. `ruff` was clean, and `pytest` gave 1318 passed and 3 skipped. Single mode needs no test of its own, because `run_single` never calls `retry_elsewhere`. The piped-input test covers the case without a terminal._

## Manual verification

- [x] 2.1 Walk through RRG-89's "done when" in a real terminal:
  - start on a provider that cannot answer
  - ask a question, record the error and the list
  - press Enter and record that nothing changes
  - ask again, pick a provider with a valid key, and record that it answers the same question

  _Evidence (Windows 11, 2026-11-10, `pruebas/demo-rrg89.py`, 8 of 8 checks, run while Nathalia watched): `nar` ran in a real Windows Terminal console with keys typed through `WriteConsoleInput`. Nathalia's real DeepSeek and Anthropic keys came from the Windows user environment. The run went as follows:_
  - _`nar --provider ollama`, with Ollama not installed, so it really cannot answer._
  - _The question failed after 4 attempts with `cannot reach Ollama at http://localhost:11434; ...`._
  - _Then `ollama can't answer. Continue with another provider?` listed `1. anthropic · claude-sonnet-5-5 (key set)`, `2. openrouter · openrouter/auto (no key: it will ask for one)` and `3. deepseek · deepseek-flash (key set)`._
  - _Enter kept `provider ollama`._
  - _The same question failed again and brought the list back. `3` printed `checking deepseek · deepseek-flash...` and `model: deepseek · deepseek-flash`, and DeepSeek answered `hola`._

  _Found while testing, already on `main` and out of scope: after a switch, the context limit stays the first provider's. It showed `/ 4,096 tokens`, Ollama's default, while on DeepSeek. Filed separately in Linear._
