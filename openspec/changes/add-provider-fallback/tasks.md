# Tasks

> Review workload: about 350 lines, mostly tests. It fits in one PR.

## [specs]

- [ ] 0.1 Link this change to Linear RRG-89 and verify the link opens `openspec/changes/add-provider-fallback/` on GitHub once merged
- [ ] 0.2 Open a PR in `nar-specs` from Linear's RRG-89 branch; verify `openspec validate add-provider-fallback --strict` passes

## [nar]

- [ ] 1.1 Create Linear's RRG-89 branch from `main`, and add `unavailable` to `ProviderError` and set it in `retry.py`. Verify with mocked HTTP:
  - 402 and Anthropic's "credit balance" 400 are marked
  - retries exhausted on 429 and 529 are marked
  - an unreachable server is marked
  - 401, 404 and other 400s are not marked
- [ ] 1.2 Add `ProviderMenu` in `src/nar/ui/provider_menu.py`. Verify with injected readers: the numbered list and its statuses, a valid pick, Enter, end of input, Ctrl+C, an unknown answer asked again, and no terminal
- [ ] 1.3 Add `retry_elsewhere` in `src/nar/cli/fallback.py` and call it from `repl.run`. Verify with fake providers in `tests/test_cli.py`:
  - a pick switches and the same request is answered by the new provider
  - Enter stays
  - a failing check shows the list again
  - a second unavailable failure offers the list without the tried providers
  - unmarked errors, single mode and piped input show no list
  - `ruff check .`, `ruff format --check .` and `pytest` are clean

## Manual verification

- [ ] 2.1 Walk through RRG-89's "done when" in a real terminal:
  - start on a provider that cannot answer
  - ask a question, record the error and the list
  - press Enter and record that nothing changes
  - ask again, pick a provider with a valid key, and record that it answers the same question
