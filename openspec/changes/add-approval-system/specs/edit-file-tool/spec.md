# Spec Delta

## ADDED Requirements

### Requirement: Edit preview
The description of an `edit_file` call made before it runs SHALL show the same unified diff that running it would report, SHALL leave the file untouched, and SHALL carry the same error message that running it would return whenever running it would fail without effect (empty `old_string`, identical strings, file missing, directory, binary, not UTF-8, text not found, text found more than once). A call whose resolved path is inside the working directory, and not inside its `.git` or `.nar` folders, SHALL be declared as a project edit.

#### Scenario: Preview equals the result
- **WHEN** `edit_file` is described for replacing `timeout = 10` with `timeout = 30` in `config.py`, and then the same call is run
- **THEN** the diff of the description and the diff of the result are identical, and the file only changed when the call was run

#### Scenario: Preview of a failing edit
- **WHEN** `edit_file` is described for an `old_string` that appears 3 times in the file
- **THEN** the description carries the error saying it was found 3 times and that more context is needed, and the file is unchanged

#### Scenario: Preview of a missing file
- **WHEN** `edit_file` is described for a path that does not exist
- **THEN** the description carries the error saying the file was not found, and no file is created
