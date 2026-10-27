# Tasks

## [specs]

- [x] 0.1 Link this change to Linear RRG-66 (move the issue to In Progress and add the `nar-specs` change path as a link) and verify the link opens `openspec/changes/add-search-tools/` on GitHub once merged
- [x] 0.2 Open a PR in `nar-specs` from Linear's RRG-66 branch with proposal, specs, design and tasks; verify `openspec validate add-search-tools --strict` passes

## [nar]

## 1. Branch

- [x] 1.1 In `nar/`, update `main` and create the branch `nathaliascode/rrg-66-add-search-tools-grep-y-glob`; verify `git status` shows the new branch and a clean tree

## 2. Shared search helpers

- [x] 2.1 Create `src/nar/tools/search.py` with `glob_matcher(pattern)` and `display_path(path, cwd)` (design.md §3–4), with `tests/test_tools_search.py`; verify `pytest tests/test_tools_search.py` passes for: `**/*.py` matches `main.py` and `src/utils/helpers.py` but not `README.md`; `src/*.py` matches `src/app.py` and not `src/utils/helpers.py`; `test_*.py` matches `tests/unit/test_b.py`; `src/**` matches `src/a/b.txt`; `?` does not match `/`; `[!a]bc` matches `xbc` and not `abc`; an unclosed `[` is literal; `./src/*.py` behaves like `src/*.py`; `display_path` gives `src/app.py` for a file under `cwd` and the absolute path for one outside it
- [x] 2.2 Add `walk_files(root)` to `src/nar/tools/search.py` (git `ls-files` first, `os.walk` fallback, `IGNORED_NAMES` pruning, sorted, design.md §2) and extend `tests/test_tools_search.py`; verify for: a non-git folder lists `src/app.py` and skips `.venv/lib/site.py` and `node_modules/x.js`; a `git init` folder with `.gitignore` `build/` skips `build/out.py` and keeps untracked `src/app.py`; a nested `.gitignore` in `src/` ignoring `*.log` is honored; a tracked file deleted from disk is not listed; with `subprocess.run` patched to raise `FileNotFoundError` the fallback walk is used; git tests are skipped when `git` is not on `PATH`

## 3. glob tool

- [x] 3.1 Create `src/nar/tools/glob.py` (`GlobTool`, design.md §4) with `tests/test_tools_glob.py`; verify `pytest tests/test_tools_glob.py` passes for every scenario in `specs/glob-tool/spec.md`: recursive pattern, `path` = `src`, single star, pattern without slash, gitignored file, noise folders outside git, 250 matches capped at 200 with the total stated, no matches, missing folder, a file as `path`, and `GlobTool().requires_approval is False`

## 4. grep tool

- [x] 4.1 Create `src/nar/tools/grep.py` (`GrepTool`, design.md §5) with `tests/test_tools_grep.py`; verify `pytest tests/test_tools_grep.py` passes for every scenario in `specs/grep-tool/spec.md`: definition found as `path:line:text`, single file, `glob` filter, `ignore_case`, invalid regex (no file read), binary file skipped, gitignored file skipped, 150 matches stopped at 100, a 5,000-character line cut to 300 plus ` [...]`, no matches, missing path; plus CRLF files give correct line numbers without `\r`, a form feed inside a line does not shift line numbers, a file over 10 MB is skipped, invalid UTF-8 is searched with replacement characters, an explicit binary file gives an error, and `GrepTool().requires_approval is False`

## 5. Registration, prompt and docs

- [x] 5.1 Register `GlobTool` and `GrepTool` after `ListDirTool` in `src/nar/tools/builtin.py` and update `tests/test_tools_registry.py`; verify `create_default_registry().names() == ["read_file", "write_file", "edit_file", "list_dir", "glob", "grep"]` and both definitions export only `name`, `description`, `parameters`
- [x] 5.2 Add the glob/grep guideline line to `GUIDELINES` in `src/nar/agent/prompt.py`; verify `pytest tests/test_system_prompt.py` passes and the built prompt contains `grep`
- [x] 5.3 Add the two tools to the README's built-in tools section (patterns, ignore rules, caps, git fallback) with `scripts/run_tool.py` examples; verify the documented commands run as written from the repo root

## 6. Delivery

- [x] 6.1 Run `ruff check .`, `ruff format --check .` and `pytest`; verify all three pass, then push the branch and open a PR in `nar` referencing RRG-66

## 7. Manual verification (RRG-66 "done when")

- [ ] 7.1 Walk through RRG-66's "done when" ("the agent finds where a function is defined in a real repo") at the `nar/` repo root with `NAR_HOME` pointing at a temp folder, and record what was observed for each step:
  - `python scripts/run_tool.py glob '{"pattern": "**/*.py", "path": "src"}'` → Python files under `src/` only, no `__pycache__` entries
  - `python scripts/run_tool.py grep '{"pattern": "def load_config"}'` → `src/nar/config/loader.py:<line>:def load_config(...`
  - `nar --provider openrouter "¿dónde se define la función load_config?"` → stderr shows a `grep` (or `glob`) call; the answer names `src/nar/config/loader.py` and the line; exit 0
  - Same question with `--provider ollama --model <local model>` (and `anthropic` when a key is available)

  Observed on 2026-10-27 (Linux, `NAR_HOME` in a temp folder, nar branch `nathaliascode/rrg-66-add-search-tools-grep-y-glob`):
  - `glob` `**/*.py` in `src` → `found 42 files`, all under `src/nar/`, no `__pycache__`/`.pyc` (`**/*` in `src` also lists none) ✔
  - `grep` `def load_config` with `glob` `*.py` → `src/nar/config/loader.py:43:def load_config(cwd: Path, overrides: ...` plus the test fixtures that mention it ✔
  - OpenRouter (`openrouter/auto`): `nar --provider openrouter "¿dónde se define la función load_config?"` → one `grep(pattern="def load_config")` call (✓ 6 matches in 4 files), answer `src/nar/config/loader.py:43`, telling the definition apart from the README and test references; 10.6 s, exit 0 ✔
  - Ollama (`qwen3.5:4b`, CPU): running; Anthropic: **pending** (no `ANTHROPIC_API_KEY`)
