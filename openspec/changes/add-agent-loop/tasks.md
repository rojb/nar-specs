# Tasks

## [specs]

- [x] 0.1 Link this change to Linear RRG-63 (move the issue to In Progress and add the `nar-specs` change path as a link) and verify the link opens `openspec/changes/add-agent-loop/` on GitHub once merged
- [x] 0.2 Open a PR in `nar-specs` from Linear's RRG-63 branch with proposal, specs, design and tasks; verify `openspec validate add-agent-loop --strict` passes
- [ ] 0.3 Before archiving this change, archive `add-llm-providers` (RRG-61) first (design.md, Context); verify `openspec/specs/cli-entrypoint/spec.md` then contains the provider scenarios of "Single mode"

## [nar]

## 1. Branch and configuration

- [ ] 1.1 In `nar/`, update `main` and create a branch from Linear's RRG-63 branch name; verify `git status` shows the new branch
- [ ] 1.2 Add `max_turns` (`int`, default 20, `>= 1`) to `NarConfig` and extend `tests/test_config.py` with the scenarios of `specs/configuration/spec.md`; verify `pytest tests/test_config.py` passes and `nar --show-config` shows `max_turns = 20  (default)`

## 2. Context and system prompt

- [ ] 2.1 Implement `nar/context/manager.py` (`ContextManager`: history, `mark`/`rollback`, usage-based token count with estimate) from design.md §4, with `tests/test_context_manager.py`; verify history order, rollback and that the count grows after a long tool result
- [ ] 2.2 Implement `nar/agent/prompt.py` (`build_system_prompt`) from design.md §5, with `tests/test_system_prompt.py`; verify it names the working directory and OS, includes `AGENTS.md` content under a project-instructions heading, and omits that section when there is none

## 3. Agent loop

- [ ] 3.1 Implement `nar/agent/events.py` and `nar/agent/loop.py` (`Agent.run`, `ToolGate`, `approval_gate`) from design.md §2, §3 and §7, with `tests/test_agent_loop.py` using a scripted fake provider; verify every scenario of `specs/agent-loop/spec.md` that does not need a terminal (final answer, calls in order, error results, turn limit with results for unrun calls, approval at `ask`/`yolo`) and the rollbacks of `specs/agent-context/spec.md` (provider error, generator closed mid-stream)

## 4. Terminal view and CLI

- [ ] 4.1 Implement `nar/ui/agent_view.py` (`AgentView`) from design.md §6, with `tests/test_agent_view.py` on `StringIO` consoles; verify text goes to `out`, tool lines, warnings and the cut-off notice go to `err`, and the answer always ends with a newline
- [ ] 4.2 Implement `nar/cli/runner.py` and rewire single mode in `nar/cli/main.py` (turn limit → exit 1, Ctrl+C → `cancelled`, exit 130); update `tests/test_cli.py` with a fake provider; verify `pytest tests/test_cli.py` passes and `nar --version` still takes well under a second
- [ ] 4.3 Rewire `nar/cli/repl.py` to the agent (conversation kept across lines, Ctrl+C or provider error during a request returns to the prompt); extend `tests/test_cli.py`; verify a two-line session where the second request refers to the first sends both to the provider
- [ ] 4.4 Update `nar/README.md` (agent behavior in both modes, `max_turns`, stdout/stderr split, Ctrl+C, approval-required tools before RRG-69); verify every command in the changed sections runs as written

## 5. Delivery

- [ ] 5.1 Run `ruff check .`, `ruff format --check .` and `pytest`; verify all three pass, then push the branch and open a PR in `nar` referencing RRG-63

## 6. Manual verification (RRG-63 "done when")

- [ ] 6.1 Walk through RRG-63's "done when" ("`nar \"lee pyproject.toml y dime la versión\"` uses `read_file` and answers correctly with any provider") at the `nar/` repo root with `NAR_HOME` pointing at a temp folder, and record what was observed for each step:
  - `nar --provider openrouter "lee pyproject.toml y dime la versión"` → `read_file` shown on stderr, answer states the version from `src/nar/__init__.py` or `pyproject.toml`, exit 0
  - Same prompt with `--provider ollama --model <local model>` and with `--provider anthropic` (when a key is available)
  - `nar ... > answer.txt` → the file holds only the answer
  - `nar` interactive: ask to read `pyproject.toml`, then "¿qué versión dijo?" → answered from the history; Ctrl+C while streaming → `cancelled`, prompt returns; `exit` → code 0
  - `max_turns = 1` in `.nar/config.toml` and a prompt that needs a tool → turn-limit message on stderr, exit 1
