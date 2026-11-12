# Design

## Context

- `runner.create_agent(config, cwd, instructions, view, *, trust_project_tools=False)` builds every agent, in interactive and single mode. It builds the system prompt once with `build_system_prompt(cwd, instructions, memory)`, creates the default registry, registers `TaskTool`, and loads custom tools last, reporting problems through `view.warn`. The system prompt is never rebuilt during a session: `/clear`, compaction, `/model` and `/resume` keep it.
- Tools are synchronous `Tool` subclasses returning `ToolResult.ok(...)` or `ToolResult.error(...)`. The framework already truncates output at 30,000 characters.
- Slash commands are `register_command(name, handler, summary)` in `cli/slash.py`; handlers receive a `SlashContext` (built positionally in `repl.run`) and cannot send a request today. Requests run only in `repl.run` through `run_request(agent, view, line, recorder)`, followed by `retry_elsewhere(...)` on a provider error.
- PyYAML is not installed, not even transitively.

## Goals / Non-Goals

**Goals:**
- Discovery and parsing live in one small package, usable without a provider, so they can be tested alone.
- The skill list is computed once per agent, like `AGENTS.md`, and the instructions are read from disk on every use.

**Non-Goals:**
- Subagents (`TaskTool`) do not get the skill list or the `skill` tool.
- No reload of the skill list during a session: a skill added after startup needs a restart.
- No completion of skill names after `/skill`.

## Decisions

### 1. Frontmatter parser: PyYAML
`pyproject.toml` (MODIFIED) adds `pyyaml>=6` to `[project].dependencies`, used through `yaml.safe_load`. Real skills use quoted, folded and multi-line descriptions; a hand-written `key: value` parser would reject valid files or accept invalid YAML, against the spec. Alternative considered: a minimal built-in parser, which avoids a dependency but only handles single-line scalars.

### 2. `nar.skills` package (NEW)
`src/nar/skills/__init__.py` and `src/nar/skills/loader.py`:
- `@dataclass(frozen=True) Skill(name: str, description: str, folder: Path, origin: Literal["project", "user"])`, with a `file` property returning `folder / "SKILL.md"`.
- `class SkillError(NarError)` carrying the file and the reason.
- `user_skills_dir() -> Path` returns `nar_home() / "skills"`; `project_skills_dir(cwd) -> Path` returns `cwd / ".nar" / "skills"`.
- `parse_skill(folder, origin) -> Skill` reads `SKILL.md` as UTF-8, splits the frontmatter (first line `---`, closing line `---`), checks every rule in the spec, and raises `SkillError` with the reason.
- `discover_skills(cwd, warn: Callable[[str], None]) -> tuple[Skill, ...]` scans the user folder, then the project folder (subfolders in name order, one level only, folders without `SKILL.md` ignored), reports each `SkillError` as one warning, lets project skills replace user skills by name, and returns the skills sorted by name. It never raises.
- `read_instructions(skill) -> str` re-reads the file and returns the text after the frontmatter, stripped; it raises `SkillError` if the file is gone, unreadable or no longer valid.

### 3. `SkillTool` (NEW)
`src/nar/tools/skill.py`: `SkillTool(skills: Sequence[Skill])`, `name = "skill"`, `requires_approval = False`, one required string parameter `name`. The result is:

```
Skill: <name>
Folder: <absolute folder path>

<instructions>
```

An unknown name returns an error listing the available names; a `SkillError` returns an error naming the file. Its description tells the model to call it before working on a request that matches a listed skill.

### 4. Wiring in `create_agent`
`src/nar/cli/runner.py` (MODIFIED): `create_agent` calls `discover_skills(cwd, view.warn)` before building the prompt, passes the skills to `build_system_prompt`, registers `SkillTool(skills)` after `TaskTool` when the tuple is not empty, and stores the tuple on the agent.

`src/nar/agent/loop.py` (MODIFIED): `Agent` gets a `skills: tuple[Skill, ...]` attribute, empty by default, so commands can reach it through `SlashContext.agent`.

`src/nar/agent/prompt.py` (MODIFIED): `build_system_prompt(..., skills: Sequence[Skill] = ())` appends a `# Skills` section after the project instructions: one line per skill (`- <name>: <description>`) and the instruction to call the `skill` tool first when a request matches. No section when the tuple is empty.

### 5. `/skills` and `/skill`
`src/nar/cli/commands.py` (MODIFIED) registers both:
- `/skills` prints `<name> (<project|user>): <description>` per skill on standard error, or the "no skills found" line naming both folders.
- `/skill <name> [request]` reads the instructions with `read_instructions` right away, stores a `PendingSkill(name, folder, instructions)` on the context, and prints `Skill <name> will be used with the next request.` With a request, it also sets `context.queued_request = request`.

`src/nar/cli/slash.py` (MODIFIED): `SlashContext` gets `pending_skill: PendingSkill | None = None` and `queued_request: str | None = None`.

`src/nar/cli/repl.py` (MODIFIED):
- After `dispatch`, if `queued_request` is set, it is taken (and cleared) and run as the line; otherwise the loop continues as today.
- Before `run_request`, when a skill is pending, the line becomes the composed text below, and the pending skill is cleared. The composed text is also the line passed to `retry_elsewhere`, so a fallback retry keeps the skill.

```
The user chose the skill "<name>" for this request. Its folder is <folder>.
Follow its instructions:

<instructions>

Request:
<line>
```

`/clear` (`commands._clear`, MODIFIED) also sets `pending_skill = None`.

No new configuration keys.

## Risks / Trade-offs

- **Pending skill consumed when the request starts** → if the user cancels or the request fails without a fallback, the skill is gone with the rolled-back request. Keeping it would mean the next unrelated request carries it unexpectedly; typing `/skill` again is cheap.
- **Hooks and context management see the composed text** → `before_agent` hooks and token estimates include the instructions. This is the true size of what is sent, so it is the correct number.
- **Large SKILL.md through `/skill`** → not truncated, unlike the tool result. Skills are meant to be short; compaction handles a large conversation as usual.
- **New dependency** → PyYAML is a mature, pure-Python-capable package with wheels for every supported platform.
- **Project skills without a trust prompt** → a hostile repository can steer the model through a skill, exactly as it can through `AGENTS.md`; every tool call still goes through approval.

## Migration Plan

None. Without skill folders, `nar` behaves exactly as before.
