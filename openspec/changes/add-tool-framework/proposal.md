# Proposal

- **Owner:** Ruben
- **Linear issue:** RRG-62
- **Video timestamps covered:** 02:30:46 (Tool calling — Base Tool), 03:09:17 (Read File Tool), 03:48:31 (Tool Registry — registering tools, passing schemas to the LLM), 10:03:11 (Displaying Error Messages on TUI)
- **Depends on:** RRG-60 `scaffold-cli` (archived)
- **Runs in parallel with:** RRG-61 `add-llm-providers` (nathaliascode)
- **Unblocks:** RRG-63 `add-agent-loop`, every Phase 2 tool (RRG-64, RRG-65, RRG-66, RRG-67), RRG-75 `add-custom-tools`, RRG-74 `add-mcp-client`
- **Course requirement covered:** none directly; it is the base that human in the loop (RRG-69) and external MCP tools (RRG-74) plug into.

## Why

An agent is only useful if the model can act on the codebase, and every action goes through a tool. Before the agent loop (RRG-63) and the concrete tools (Phase 2) can be built, `nar` needs one shared contract for what a tool is, how its arguments are validated, how its result and errors are reported back to the model, and how tool calls look in the terminal. Building it now, in parallel with the providers, keeps the two halves of the loop independent.

## What Changes

- Introduce a **tool contract**: every tool has a unique name, a description for the model, a JSON Schema for its parameters, a flag saying whether it needs user approval, and an execution that returns a structured result (success with output, or error with a message).
- Validate tool arguments against the schema **before** running the tool; invalid arguments return an error result to the model instead of crashing `nar`.
- Truncate very long tool output to a fixed budget with a visible marker, so one tool call cannot flood the context.
- Add a **tool registry** that registers tools by name, rejects duplicate names, looks tools up, and exports all tool definitions in a **provider-neutral** shape (name, description, parameters schema). Converting that shape to Anthropic/OpenAI formats is left to the provider adapters of RRG-61, so neither change depends on the other.
- Add the first built-in tool, **`read_file`**: reads a text file relative to the working directory, with optional line range, line numbers in the output, and clear errors for missing, binary or oversized files.
- Render tool calls, their results and their errors in the terminal (tool name, short argument summary, outcome), introducing Rich as the rendering library.
- Add a developer script to run any registered tool by name with JSON arguments, so tools can be verified before the agent loop exists (the video uses the same approach with `scripts/test_tool.py`).

Out of scope: calling tools from an LLM (RRG-63), asking the user for approval (RRG-69 only reads the flag defined here), the remaining tools (Phase 2), loading user-defined tools (RRG-75), MCP tools (RRG-74).

## Capabilities

### New Capabilities

- `tool-framework`: the tool contract, argument validation, result and error reporting, output truncation, the tool registry and its provider-neutral definitions export, and how tool activity is shown to the user.
- `read-file-tool`: the behavior of the built-in `read_file` tool.

### Modified Capabilities

<!-- None: cli-entrypoint, configuration and project-instructions do not change. -->

## Impact

- **Code (`nar` repo):** new `src/nar/tools/` package (base contract, registry, built-in `read_file`), new `src/nar/ui/` module for Rich rendering of tool activity, a `ToolError` added to `src/nar/errors.py`, a `scripts/run_tool.py` developer script.
- **Dependencies:** `rich` (rendering) and `jsonschema` (argument validation) — final choice confirmed in design.
- **Interfaces other changes rely on:** the tool contract and registry API (RRG-63 calls them; RRG-64..67, RRG-75 and RRG-74 add tools through them); the provider-neutral definition shape (RRG-61 converts it); the approval flag (RRG-69 reads it).
- **User-visible:** no new `nar` command or flag; tool activity rendering becomes visible once RRG-63 connects the loop.
