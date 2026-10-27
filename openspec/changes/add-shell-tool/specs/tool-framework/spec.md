# Spec Delta

## MODIFIED Requirements

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
