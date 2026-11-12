# Spec Delta

## ADDED Requirements

### Requirement: Language option
The `nar` command SHALL accept a `--language <code>` option in both modes, choosing the language the model replies in (`es` or `en`) for that run; `--help` SHALL list it together with the allowed values.

#### Scenario: Default reply language
- **WHEN** the user runs `nar "what does src/app.py do?"` with no configuration and no option
- **THEN** the model is told to reply in Spanish, and the answer is in Spanish even though the question is in English

#### Scenario: English from the command line
- **WHEN** the user runs `nar --language en "que hace src/app.py?"`
- **THEN** the model is told to reply in English, whatever the configuration files say, and the answer is in English

#### Scenario: Invalid language
- **WHEN** the user runs `nar --language fr "hello"`
- **THEN** `nar` exits with a non-zero code and the error lists `es` and `en`

#### Scenario: Help lists the option
- **WHEN** the user runs `nar --help`
- **THEN** the output lists `--language` with the two allowed values
