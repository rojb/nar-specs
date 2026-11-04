# custom-tools Specification

## Purpose
TBD - created by archiving change add-custom-tools. Update Purpose after archive.

## Requirements

### Requirement: Discovering custom tools
At startup, in interactive and single mode, `nar` SHALL look for `.py` files directly in `tools/` of the nar folder (`~/.nar`, or `NAR_HOME`) and in `.nar/tools/` of the working directory, in name order, skipping files whose name starts with `_`. Every non-abstract subclass of `Tool` defined in a file SHALL be created without arguments and registered, so the model can call it as any other tool. Classes imported into a file from elsewhere SHALL NOT be registered. When a tool loads, `nar` SHALL print a line on standard error naming it. Loading SHALL NOT leave files in the tools folder. With no tool files nothing SHALL be printed.

#### Scenario: Tool in the global folder
- **WHEN** `~/.nar/tools/word_count.py` defines a `Tool` subclass named `word_count`
- **THEN** `nar` registers it without asking, and the model can call it

#### Scenario: Tool in the project folder
- **WHEN** `.nar/tools/word_count.py` defines a `Tool` subclass and the user confirmed that project's tools
- **THEN** the model can call the tool without any change to `nar`'s code

#### Scenario: Helper files and imported classes
- **WHEN** a folder has `_helper.py`, and a tool file imports `ReadFileTool`
- **THEN** `_helper.py` is not loaded and `ReadFileTool` is not registered again

#### Scenario: Same folder twice
- **WHEN** the working directory is the user's home, so both folders are the same one
- **THEN** its files are loaded once, as global tools

### Requirement: Confirmation for project tools
`nar` SHALL NOT run any file from `.nar/tools/` until the user has allowed that folder with those exact contents. Allowed means one of: a remembered answer whose folder and fingerprint of file names and contents match, a "yes" to the question, or `--trust-project-tools` on the command line. The question SHALL be asked on standard error, SHALL list the files, and SHALL say that they run as code with the user's permissions. A "yes" SHALL be remembered in `trusted_tools.json` in the nar folder, and so SHALL the flag. A "no", end of input or Ctrl+C at the question SHALL skip the project tools for this run, with a line saying so, without remembering anything and without ending `nar`. No setting in a configuration file and no approval level SHALL allow project tools.

#### Scenario: First time
- **WHEN** the user runs `nar` in a project with `.nar/tools/a.py` and answers yes
- **THEN** the tool is available, and the next run in that project does not ask

#### Scenario: Refused
- **WHEN** the user answers no
- **THEN** the file is not run, its tools are not available, and `nar` continues; the next run asks again

#### Scenario: A file changes
- **WHEN** a previously allowed project adds a tool file or changes one
- **THEN** `nar` asks again before running any of them

#### Scenario: Cannot ask
- **WHEN** standard input or standard error is not a terminal and the project was not allowed
- **THEN** the files are not run, a warning names `--trust-project-tools`, and the run continues with exit code 0

#### Scenario: Configuration cannot allow it
- **WHEN** the project's `.nar/config.toml` sets `approval = "yolo"` and the project was not allowed
- **THEN** `nar` still asks, and it does not run the files until the user answers yes

#### Scenario: Global tools never ask
- **WHEN** only the global folder has tool files
- **THEN** no question is asked, also without a terminal

### Requirement: Failures never break nar
A tool file that cannot be loaded SHALL produce one warning naming the file and the reason, and the other files SHALL still load. This SHALL cover a syntax or import error, an exception while the file runs, `sys.exit`, a file that defines no tool, a class that cannot be created without arguments, and a tool the registry rejects (bad name or schema). Ctrl+C SHALL still interrupt `nar`. A remembered-answers file that is not valid SHALL count as nothing remembered, with a warning, and SHALL be replaced only when the user allows a project.

#### Scenario: Broken file next to a good one
- **WHEN** one file has a syntax error and another defines a valid tool
- **THEN** a warning names the first, and the second tool is available

#### Scenario: File calls sys.exit
- **WHEN** a tool file calls `sys.exit(1)` while it loads
- **THEN** a warning names it and `nar` keeps running

### Requirement: Names cannot be taken over
Tools SHALL be registered in the order built-in, global, project. A custom tool whose name is already registered SHALL NOT replace the existing tool, and SHALL produce a warning naming the file and the tool.

#### Scenario: Shadowing a built-in
- **WHEN** a project tool is named `read_file`
- **THEN** the built-in `read_file` stays, and a warning names the file

### Requirement: Approval of custom tools
A custom tool SHALL go through the approval system according to its own `requires_approval`, like a built-in tool. Being allowed to load SHALL NOT approve its calls.

#### Scenario: Tool that requires approval
- **WHEN** a custom tool sets `requires_approval = True` and the level is `ask`
- **THEN** each call asks the user, as for `write_file`

### Requirement: Example and guide
The `nar` repository SHALL include a working example tool, and its README SHALL explain where to put a tool file, how to write it, how confirmation works, and that a custom tool runs with the user's permissions.

#### Scenario: Example works
- **WHEN** the example file is copied to the global tools folder
- **THEN** its tool is registered and returns a correct result
