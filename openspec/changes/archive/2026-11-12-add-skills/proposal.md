# Proposal

- **Owner:** Ruben
- **Linear issue:** RRG-92
- **Video timestamps covered:** none (asked for by Ruben on 2026-11-12)
- **Depends on:** none (builds on `project-instructions` and `custom-tools`, both merged)
- **Course requirement covered:** context management (instructions are loaded only when they are needed)

## Why

The only way to give `nar` reusable knowledge today is `AGENTS.md`, and it is loaded in full in every session. There is no way to keep specialized instructions, such as how to write a migration or how to cut a release, that the agent reads only when a request needs them, and no way to share them across projects.

## What Changes

- At startup, `nar` finds skills: folders that hold a `SKILL.md` file, in `skills/` of the nar folder (`~/.nar`, or `NAR_HOME`) and in `.nar/skills/` of the working directory. When both define the same name, the project skill wins.
- A `SKILL.md` starts with YAML frontmatter that has `name` and `description`, followed by Markdown instructions. The folder may hold other files that the instructions refer to.
- The system prompt lists only the name and description of each skill. The instructions are read when a skill is used.
- A new `skill` tool lets the model use a skill by name. It returns the instructions and the path of the skill's folder.
- `/skill <name>` uses a skill from the prompt, and `/skills` lists the skills found with where each came from.
- A `SKILL.md` that cannot be read, has no frontmatter, or lacks `name` or `description` produces one warning naming the file and is skipped. `nar` still starts.
- Skills are read as text, like `AGENTS.md`. `nar` never runs a file from a skill folder by itself, so project skills need no trust prompt.

Out of scope: installing skills from a remote registry, running a skill's scripts automatically, and limiting which tools a skill may use.

**"Done when" (RRG-92):** with a skill in `.nar/skills/demo/SKILL.md`:
- `/skills` lists it;
- the model uses it through the `skill` tool when a request matches its description;
- `/skill demo` uses it from the prompt;
- a broken `SKILL.md` shows a warning and does not stop startup.

## Capabilities

### New Capabilities

- `skills`: where skills are found, the `SKILL.md` format, precedence between user and project skills, handling of broken files, and the `skill` tool.

### Modified Capabilities

- `agent-context`: the system prompt also lists the available skills by name and description. MODIFIED.
- `tui-commands`: new `/skill` and `/skills` commands. ADDED.

## Impact

- **Code (`nar` repo):**
  - `src/nar/skills/` (new), for discovery and parsing
  - `src/nar/tools/skill.py` (new), for the `skill` tool
  - `src/nar/agent/prompt.py`, to list skills in the system prompt
  - `src/nar/cli/commands.py` and `src/nar/cli/repl.py`, for `/skill`, `/skills` and the startup warnings
  - README and tests
- **Dependencies:** possibly a YAML parser for the frontmatter. `design.md` decides between adding one and a small built-in parser.
