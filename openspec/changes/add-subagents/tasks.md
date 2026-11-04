# Tasks

> Review workload: about 400 lines of code and docs plus 650 lines of tests, above the ~400-line guide (a planning heuristic, not a cap; tests are never dropped to fit). Split into two stacked PRs (design.md §9): PR A = groups 2 and 3, PR B = groups 4 and 5. Retarget the child to `main` before merging the parent with branch deletion. Following the README order, code starts after these artifacts are approved.

## [specs]

- [x] 0.1 Link this change to Linear RRG-73 (move the issue to In Progress and add the `nar-specs` change path as a link) and verify the link opens `openspec/changes/add-subagents/` on GitHub once merged
- [x] 0.2 Write `docs/SUBAGENTES.md` (what subagents are, why they isolate context, how Claude Code, Codex and OpenCode do it, and what was decided), from their official documentation
- [x] 0.3 Open a PR in `nar-specs` from Linear's RRG-73 branch with the research, proposal, specs, design and tasks; verify `openspec validate add-subagents --strict` passes

## [nar]

## 1. Branch

- [x] 1.1 In `nar/`, update `main` and create `nathaliascode/rrg-73-1-task-tool`; verify a clean tree and record which tests already fail

## 2. Foundations (PR A)

- [x] 2.1 Add `subagent_max_turns` (design.md §2), extending `tests/test_config.py`; verify every scenario of `specs/configuration/spec.md`
- [x] 2.2 Add `ToolRegistry.subset` (design.md §4), extending `tests/test_tools_registry.py`; verify it keeps the context and the instances, in the order asked, and that a missing name is skipped
- [x] 2.3 Add `build_subagent_prompt` and the main guideline (design.md §5), extending `tests/test_system_prompt.py`; verify the role, environment and project instructions are present, and memory, `todos` guidance and the main guidelines are not

## 3. The task tool (PR A)

- [x] 3.1 Create `src/nar/agent/subagent.py` (design.md §3, §4 and §6) with `tests/test_subagent.py` using a scripted provider; verify the scenarios of "Task tool", "Isolated context", "Subagent types", "Limits" and "Result of a subagent", including that the first subagent call has only the prompt, the main conversation grows by exactly one call and one result, and that an `explorer` asking for `write_file`, `shell` or `task` gets an unknown tool error with nothing changed
- [x] 3.2 Pass the gate and hooks (design.md §7), extending `tests/test_subagent.py`; verify every scenario of "Approvals and hooks in a subagent" with the real `ApprovalGate` and `HookRunner`

## 4. CLI and display (PR B)

- [x] 4.1 Register `TaskTool` in `create_agent` and show progress (design.md §8), extending `tests/test_cli.py` and `tests/test_agent_view.py`; verify `task` reaches the model, a `general` sees MCP and custom tools, standard output holds only the main answer, the waiting line stops before a question, and a saved session has no subagent messages

## 5. Docs and checks (PR B)

- [x] 5.1 Document subagents in the README, with the types, the limit, the cost in tokens, that hooks also fire for them, and a link to `docs/SUBAGENTES.md`
- [x] 5.2 Run `ruff check .`, `ruff format --check .` and `pytest`; verify the only failure, if any, is the known Linux one, then push and open the stacked PRs referencing RRG-73

## 6. Manual verification

- [ ] 6.1 Walk through RRG-73's "done when" ("explore this repo and explain the architecture" launches an `explorer`, and the main agent's context only grows with the summary) at the `nar/` repo root with `NAR_HOME` pointing at a temp folder, in a real terminal: with OpenRouter, send the request, record the `task` call and the result line; open the session file and record that the messages are the request, the call, the summary and the answer, with no file contents; compare the usage line with a run of the same request with `task` unavailable, if the model delegates; ask a `general` to change a file and answer `n`, and record that the file did not change; redirect standard output and record that the file holds only the answer. Ollama is not run, at Nathalia's request (2026-11-03). Anthropic is pending until the team has an `ANTHROPIC_API_KEY` (decision of 2026-10-27)

  _Evidence (OpenRouter, `openrouter/auto`, temporary `NAR_HOME`, a real terminal through a pty, 2026-11-04): "Explora este repo y explícame la arquitectura en 5 líneas" launched an `explorer` unprompted, shown as `explorer: 23 tool calls, 4 turns`. The session file held the request, one `list_dir` of the main agent, the `task` call, a summary of 11,883 characters and the answer, with no subagent messages; the usage line ended at 6,992 tokens. The same request told not to use `task` made 12 tool calls and ended at 22,438 tokens. These are single runs, not a benchmark, and the tokens of the subagent's own calls were not measured. A `general` asked to create a file asked first, and after `n` the file was not created. With standard output redirected, the file held only the answer, which was correct. The progress line showed `explorer: 2 tool calls...`._

  _Not run with Ollama, at Nathalia's request (2026-11-03). Still open: Anthropic._
