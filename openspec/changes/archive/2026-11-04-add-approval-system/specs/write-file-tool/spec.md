# Spec Delta

## ADDED Requirements

### Requirement: Write preview
The description of a `write_file` call made before it runs SHALL state the resolved path and whether the file would be created or overwritten, SHALL leave the file system untouched (no file and no parent folder is created), and SHALL show the change: a diff against the current content when the file exists and is readable text, or the beginning of the new content when the file is new or the existing file cannot be shown as text. It SHALL carry the same error message that running the call would return whenever running it would fail without effect (the path is a directory, a parent is not a folder, the content cannot be encoded). A call whose resolved path is inside the working directory, and not inside its `.git` or `.nar` folders, SHALL be declared as a project edit.

#### Scenario: Preview of a new file
- **WHEN** `write_file` is described for `docs/guide.md` that does not exist, in a folder without `docs`
- **THEN** the description says the file would be created, shows the beginning of the content, and neither `docs` nor `guide.md` exists afterwards

#### Scenario: Preview of an overwrite
- **WHEN** `write_file` is described for an existing text file with different content
- **THEN** the description says the file would be overwritten and shows a diff from the current content to the new content, and the file is unchanged

#### Scenario: Preview when the target is a directory
- **WHEN** `write_file` is described for the path of a directory
- **THEN** the description carries the error saying the path is a directory

#### Scenario: Repository and configuration folders
- **WHEN** `write_file` is described for `.git/hooks/pre-commit` or `.nar/config.toml` inside the working directory
- **THEN** the call is not declared as a project edit

#### Scenario: Path outside the working directory
- **WHEN** `write_file` is described for an absolute path outside the working directory
- **THEN** the call is not declared as a project edit
