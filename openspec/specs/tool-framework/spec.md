# tool-framework Specification

## Purpose
Defines what a tool is in `nar`, how tool calls are validated, executed and reported back to the model, how tools are registered and exported to LLM providers, and how tool activity is shown to the user.

## Requirements

### Requirement: Tool definition
Every tool SHALL expose a name, a description written for the model, a JSON Schema (type `object`) describing its parameters, and whether it requires user approval before running. Tool names SHALL match `^[a-zA-Z][a-zA-Z0-9_]{0,63}$` so they are accepted by every supported provider.

#### Scenario: Tool exposes all fields
- **WHEN** the built-in `read_file` tool is inspected
- **THEN** it exposes the name `read_file`, a non-empty description, a parameters schema of type `object`, and `requires_approval` set to false

#### Scenario: Invalid tool name
- **WHEN** a tool named `read-file!` is registered
- **THEN** registration fails with an error naming the invalid tool name

### Requirement: Tool registry
`nar` SHALL keep a registry of available tools where tools are registered by name and looked up by name. Registering a second tool with an existing name SHALL fail. Looking up an unknown name SHALL report that the tool does not exist.

#### Scenario: Duplicate name
- **WHEN** a tool is registered with a name that is already registered
- **THEN** registration fails with an error naming the duplicated tool and the registry keeps the original tool

#### Scenario: Unknown tool requested
- **WHEN** the model requests a tool named `delete_everything` that is not registered
- **THEN** the call produces an error result saying the tool does not exist and listing the available tool names, and `nar` keeps running

### Requirement: Provider-neutral tool definitions export
The registry SHALL export the definitions of all registered tools as a list of objects with exactly the fields `name`, `description` and `parameters` (the JSON Schema), in registration order, independent of any LLM provider format.

#### Scenario: Export
- **WHEN** the registry contains `read_file` and the export is requested
- **THEN** the result is a list with one object whose `name` is `read_file`, whose `description` is the tool description and whose `parameters` is its JSON Schema

### Requirement: Argument validation
Before running a tool, its arguments SHALL be validated against its parameters schema. Invalid arguments SHALL NOT run the tool and SHALL produce an error result that names each invalid or missing argument so the model can correct the call.

#### Scenario: Missing required argument
- **WHEN** `read_file` is called without `path`
- **THEN** the tool does not run and the error result says that `path` is required

#### Scenario: Wrong argument type
- **WHEN** `read_file` is called with `offset` set to `"ten"`
- **THEN** the tool does not run and the error result names `offset` and the expected type

### Requirement: Structured tool results
Every tool call SHALL produce a result that is either a success carrying text output for the model, or an error carrying a message for the model. A failure inside a tool (including an unexpected exception) SHALL become an error result and SHALL NOT terminate `nar`.

#### Scenario: Tool raises unexpectedly
- **WHEN** a tool raises an unexpected exception while running
- **THEN** the call produces an error result containing the exception type and message, and `nar` keeps running

### Requirement: Output truncation
Tool output sent to the model SHALL be limited to 30,000 characters. Longer output SHALL be cut and end with a marker stating that it was truncated and how many characters were omitted.

#### Scenario: Very long output
- **WHEN** a tool produces 50,000 characters of output
- **THEN** the result contains the first part of the output within the limit followed by a marker saying 20,000 or more characters were omitted

### Requirement: Tool activity display
When a tool is called, the user SHALL see the tool name and a one-line summary of its arguments; when it finishes, the user SHALL see whether it succeeded or failed. Errors SHALL be visually distinct from successes and show the error message; when an error result carries a short summary, the user SHALL see that summary instead of the full message, while the model still receives the full message. When a successful result carries a diff of a file change, the user SHALL also see that diff, with added lines and removed lines visually distinct from each other and from unchanged context lines.

#### Scenario: Successful call shown
- **WHEN** `read_file` runs successfully on `pyproject.toml`
- **THEN** the terminal shows a line with `read_file` and `pyproject.toml`, followed by a success indicator and a short summary of the output (such as the number of lines read)

#### Scenario: Failed call shown
- **WHEN** a tool call ends in an error result
- **THEN** the terminal shows the tool name with an error indicator in a distinct color and the error message

#### Scenario: Failed call with a summary
- **WHEN** a tool call ends in an error result that carries the summary `timed out after 5 s` and a long message with the output captured before the failure
- **THEN** the terminal shows the error indicator and `timed out after 5 s`, and does not show the long message

#### Scenario: Diff shown for a file change
- **WHEN** `edit_file` succeeds and its result carries a diff that removes `timeout = 10` and adds `timeout = 30`
- **THEN** the terminal shows the success line and the diff below it, with the removed line and the added line in different colors and unchanged context lines left uncolored

#### Scenario: No diff for results without one
- **WHEN** a tool succeeds and its result carries no diff, such as `read_file`
- **THEN** the terminal shows only the call line and the success summary, with no diff

#### Scenario: Diff text is shown literally
- **WHEN** a diff contains text that looks like terminal markup, such as `[red]`
- **THEN** the terminal shows that text literally and does not interpret it

### Requirement: Developer tool runner
The `nar` repository SHALL include a developer script that runs any registered tool by name with arguments given as JSON and prints its result, so tools can be exercised without an LLM.

#### Scenario: Run a tool from the script
- **WHEN** a developer runs the script with tool name `read_file` and arguments `{"path": "pyproject.toml", "limit": 5}`
- **THEN** the tool activity display is shown and the first five lines of `pyproject.toml` are printed

#### Scenario: Invalid JSON arguments
- **WHEN** the arguments given to the script are not valid JSON
- **THEN** the script prints an error explaining the JSON problem and exits with a non-zero code
