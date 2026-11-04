# Tasks

> Review workload: about 300 lines of code and docs plus 350 lines of tests, near the ~400-line guide (a planning heuristic, not a cap; tests are never dropped to fit). No stacked PRs are needed: it fits in one PR. Following the README order, code starts after these artifacts are approved.

## [specs]

- [x] 0.1 Link this change to Linear RRG-75 (move the issue to In Progress and add the `nar-specs` change path as a link) and verify the link opens `openspec/changes/add-custom-tools/` on GitHub once merged
- [x] 0.2 Open a PR in `nar-specs` from Linear's RRG-75 branch with proposal, specs, design and tasks; verify `openspec validate add-custom-tools --strict` passes

## [nar]

## 1. Branch

- [x] 1.1 In `nar/`, update `main` and create `nathaliascode/rrg-75-add-custom-tools-descubrir-herramientas-propias-del-usuario`; verify a clean tree and record which tests already fail

## 2. Loading

- [x] 2.1 Create `find_tool_files`, `load_tool_file` in `src/nar/tools/custom.py` (design.md §2) with `tests/test_tools_custom.py`; verify skipped `_` files, name order, imported classes and abstract classes not counted, every failure of "Failures never break nar" as one warning with Ctrl+C still propagating, two files with the same name in different folders, and no `__pycache__` left

## 3. Trust

- [x] 3.1 Add `fingerprint` and `TrustStore` (design.md §3), extending `tests/test_tools_custom.py`; verify a change of content, a new file and a renamed file change the fingerprint, an invalid file counts as nothing remembered and is not overwritten until a project is allowed, and the file mode is 0600 on Linux
- [x] 3.2 Create `src/nar/ui/trust_prompt.py` (design.md §4) with `tests/test_trust_prompt.py`; verify it lists the files, an empty or unknown answer asks again, end of input and Ctrl+C are a no, and it is not available without terminals
- [x] 3.3 Add `load_custom_tools` (design.md §4), extending `tests/test_tools_custom.py`; verify every scenario of "Confirmation for project tools", asserting with a file that writes a marker that a refused or unasked file never runs, that `approval = "yolo"` in the project's configuration does not allow it, and the order and warnings of "Names cannot be taken over"

## 4. CLI, example and docs

- [x] 4.1 Load custom tools in `create_agent` and add `--trust-project-tools`, extending `tests/test_cli.py`; verify a missing API key fails before any tool file runs, a tool reaches the model's definitions in both modes, and a `requires_approval` tool goes through the gate
- [x] 4.2 Add `examples/tools/word_count.py` with a test that loads it and checks its result, and document custom tools in the README
- [x] 4.3 Run `ruff check .`, `ruff format --check .` and `pytest`; verify the only failure, if any, is the known Linux one, then push and open the PR referencing RRG-75

## 5. Manual verification

- [ ] 5.1 Walk through RRG-75's "done when" ("dropping a `.py` in `.nar/tools/` lets the agent use that tool without touching `nar`'s code") in a temp project with `NAR_HOME` pointing at a temp folder and a real terminal: with OpenRouter, copy the example to `.nar/tools/`, run `nar`, read the question and answer yes, ask the model to count the words of a file, and record that it called the tool; run again and record that it does not ask; change the file and record that it asks again; answer no and record that the file did not run; run with a pipe and record the warning. Ollama is not run, at Nathalia's request (2026-11-03). Anthropic is pending until the team has an `ANTHROPIC_API_KEY` (decision of 2026-10-27)

  _Evidence (OpenRouter, temporary `NAR_HOME` and project, a real terminal through a pty, 2026-11-04): with the example in `.nar/tools/`, `nar` listed the file and asked; after `y` it printed `custom tool loaded: word_count`, and the model called `word_count(path="notas.txt")` and answered 5 words, which is correct. The next run did not ask. After changing the file it asked again; after `n` it printed `project tools skipped`, and a marker the file writes when it runs was not created. With no terminal and `approval = "yolo"` in the project's configuration, the marker was not created, a warning named `--trust-project-tools`, and the exit code was 0. No `__pycache__` was left in the folder. `trusted_tools.json` has mode 0600._

  _Not run with Ollama, at Nathalia's request (2026-11-03). Still open: Anthropic._
