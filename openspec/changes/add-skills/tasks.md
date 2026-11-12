# Tasks

> Review workload: about 600 lines, more than half tests. It fits in one PR, split into one commit per group.

## [specs]

- [ ] 0.1 Link this change to Linear RRG-92 and verify the link opens `openspec/changes/add-skills/` on GitHub once merged
- [ ] 0.2 Open a PR in `nar-specs` from Linear's RRG-92 branch; verify `openspec validate add-skills --strict` passes

## [nar]

- [x] 1.1 Create Linear's RRG-92 branch from `main`, add `pyyaml>=6` to `pyproject.toml`, and verify `pip install -e .[dev]` makes `import yaml` work in the venv _Evidence: `cb2b3b1`; `import yaml` gives 6.0.3 in the venv._
- [x] 1.2 Add `src/nar/skills/` (`Skill`, `SkillError`, `parse_skill`, `discover_skills`, `read_instructions`). Verify in `tests/test_skills.py` with `NAR_HOME` and a project in `tmp_path`:
  - user and project skills are found, with their origin; `NAR_HOME` replaces `~/.nar`
  - folders without `SKILL.md`, loose files and deeper folders are ignored silently; missing folders give no warning
  - every broken case in the spec gives exactly one warning naming the file, and the other skills still load
  - extra frontmatter keys are ignored; quoted and multi-line descriptions work
  - a project skill replaces a user skill with the same name, silently; the result is sorted by name
  - `read_instructions` returns the body and raises `SkillError` once the file is deleted

  _Evidence: `6d3aaab`, 35 tests in `tests/test_skills.py`._
- [x] 1.3 Add `SkillTool` in `src/nar/tools/skill.py`. Verify in `tests/test_tools_skill.py`: the result holds the instructions and the absolute folder path, no approval is required, an unknown name lists the available names, and a deleted file gives an error naming it _Evidence: `5688160`, 7 tests in `tests/test_tools_skill.py`._
- [x] 1.4 List skills in `build_system_prompt` and wire discovery into `create_agent` (warnings through `view.warn`, `SkillTool` only when skills exist, `Agent.skills`). Verify in `tests/test_system_prompt.py` and the runner tests: the section lists names and descriptions and never the instructions, there is no section and no `skill` tool without skills, and a broken skill warns without stopping startup _Evidence: `ec5c614`, 5 tests in `tests/test_system_prompt.py` and 4 in `tests/test_cli.py`._
- [x] 1.5 Add `/skills` and `/skill` (`PendingSkill`, `queued_request`, composition in `repl.run`, `/clear` dropping a pending skill). Verify in `tests/test_slash.py` and `tests/test_cli.py` with a fake provider:
  - `/skills` lists with origin, or names both folders when empty
  - `/skill demo` then a request sends one user message holding the instructions and the request
  - `/skill demo <request>` sends at once
  - an unknown name lists the skills and adds nothing
  - `/clear` drops a pending skill
  - all output goes to standard error

  _Evidence: `70a1919`, 10 tests in `tests/test_slash.py` and 8 in `tests/test_cli.py`. No test drives the fallback retry with a skill; it reuses the composed line passed to `retry_elsewhere`._
- [x] 1.6 Document skills in the README (a `## Skills` section modeled on `## Custom tools`, `/skills` and `/skill` in `## Commands`, `skill` in `### Built-in tools`) and add an example skill under `examples/`. Verify `ruff check .`, `ruff format --check .` and `pytest` are clean _Evidence: `cff4bca`, example in `examples/skills/changelog/`. `ruff check .` passed, `ruff format --check .` reported 156 files already formatted, and `pytest` gave 1390 passed and 3 skipped (baseline 1318 passed and 3 skipped)._

## Manual verification

- [ ] 2.1 Walk through RRG-92's "done when" in a real terminal with `.nar/skills/demo/SKILL.md`:
  - run `nar`, type `/skills`, and record that `demo` is listed as project
  - ask something that matches the description, and record the `skill` tool call and an answer that follows the instructions
  - type `/skill demo` and a request, and record that the answer follows the instructions
  - break the frontmatter of a second skill, restart, and record the warning and that `nar` still starts

  _Partial evidence (Windows 11, 2026-11-12, piped input, `--provider ollama`, `NAR_HOME` and project in a temp folder): startup printed `skipping skill: ...\broken\SKILL.md: the frontmatter is missing (the file must start with ---)` and went on; `/skills` printed `demo (project): Use when the user asks for a secret greeting.`; `/skill nope` printed `error: there is no skill named 'nope'; available skills: demo`; `/skill demo` printed `Skill demo will be used with the next request.`; `/clear` worked; `/tools` listed `skill`. Pending: the two steps that need a model answer (no provider key was available in this environment); they are covered with a fake provider in `tests/test_cli.py`._
