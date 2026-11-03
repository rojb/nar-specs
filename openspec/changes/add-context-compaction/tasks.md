# Tasks

> Review workload: about 450 lines of code and docs plus 700 lines of tests, above the ~400-line guide (a planning heuristic, not a cap; tests are never dropped to fit). Split into three stacked PRs in this order: PR A = groups 2 and 3, PR B = groups 4 and 5, PR C = groups 6 and 7. For each parent, **first** retarget its child to `main` (`gh pr edit <child> --base main`), **then** merge the parent with branch deletion.

## [specs]

- [x] 0.1 Link this change to Linear RRG-70 (move the issue to In Progress and add the `nar-specs` change path as a link) and verify the link opens `openspec/changes/add-context-compaction/` on GitHub once merged
- [ ] 0.2 Open a PR in `nar-specs` from Linear's RRG-70 branch with proposal, specs, design and tasks; verify `openspec validate add-context-compaction --strict` passes

## [nar]

## 1. Branch

- [ ] 1.1 In `nar/`, update `main` and create `nathaliascode/rrg-70-1-pruning`; verify a clean tree and record which tests already fail before any change

## 2. Settings (PR A)

- [ ] 2.1 Add `context_window`, `context_threshold` and `DEFAULT_CONTEXT_WINDOWS` to `src/nar/config/models.py` (design.md §2), extending `tests/test_config.py`; verify every scenario of `specs/configuration/spec.md`, including the source shown by `--show-config`

## 3. Snapshots and pruning (PR A)

- [ ] 3.1 Change `mark`/`rollback` to snapshots and add `replace_messages` and `overhead_tokens` to `ContextManager` (design.md §3), extending `tests/test_context_manager.py`; verify a rollback restores messages replaced after the mark and the reported usage, and that `replace_messages` makes the count an estimate that includes the overhead
- [ ] 3.2 Create `src/nar/context/pruning.py` (design.md §4) with `tests/test_context_pruning.py`; verify the oldest results become the marker naming the tool, the newest stay inside the budget, short results and non-tool messages do not change, `tool_call_id` and `is_error` are kept, and a second run changes nothing

## 4. Compaction (PR B)

- [ ] 4.1 Create `src/nar/context/compaction.py` (design.md §5) with `tests/test_context_compaction.py`; verify the tail never starts with a tool result and respects both budgets, the summary message has the header, the current request is added only when it is not in the tail, an earlier summary reaches the new transcript, long transcripts are cut in the middle, the call has no tools, an empty summary raises and leaves the history, and `minimum_tokens` skips the model call

## 5. Agent loop and display (PR B)

- [ ] 5.1 Add the events and `_manage_context` (design.md §6), extending `tests/test_agent_loop.py`; verify below the limit nothing changes, pruning alone makes no summary call, compaction happens between tool calls of one request with the request text still sent and every kept call answered, a provider error warns and continues, too-small stays silent, Ctrl+C after compaction restores the history, and an agent without a compactor behaves as before
- [ ] 5.2 Show the events in `AgentView` (design.md §7), extending `tests/test_agent_view.py`; verify every line goes to standard error and nothing reaches standard output

## 6. CLI (PR C)

- [ ] 6.1 Build the `Compactor` in `create_agent` and add `compact_now` and the indicator in `repl.run`, extending `tests/test_cli.py`; verify the indicator appears after a request in interactive mode on standard error only, and not in single mode
- [ ] 6.2 Register `/compact` (design.md §8), extending `tests/test_slash.py`; verify every scenario of "Manual compaction": compacts and shows the sizes, empty conversation does not call the model, an argument is an error, a provider failure and Ctrl+C keep the history and the session

## 7. Docs and checks (PR C)

- [ ] 7.1 Document pruning, compaction, the indicator, `/compact` and the two settings in the README
- [ ] 7.2 Run `ruff check .`, `ruff format --check .` and `pytest`; verify the only failure, if any, is one recorded in 1.1, then push and open the stacked PRs referencing RRG-70

## 8. Manual verification

- [ ] 8.1 Walk through RRG-70's "done when" ("a long session keeps working after going over the model's context window") at the `nar/` repo root with `NAR_HOME` pointing at a temp folder and a small `context_window` in a temporary project config: with OpenRouter, run `nar` interactively, ask it to read large files until the indicator passes the limit and record the pruning line; continue until a compaction happens and record the sizes; ask about something from the summarized part and record the answer; run `/compact` and record the sizes; press Ctrl+C during a request and confirm the next request works. Repeat a shorter run with Ollama. Anthropic is pending until the team has an `ANTHROPIC_API_KEY` (decision of 2026-10-27)
