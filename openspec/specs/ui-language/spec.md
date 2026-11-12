# ui-language Specification

## Purpose
Defines the language of the `nar` interface: which texts the user sees in Spanish or English, which texts never change, how the language is chosen, and how prompts and output before the configuration is loaded follow it.

## Requirements

### Requirement: Catalog and active language
Every text that `nar` shows to the user in the terminal SHALL come from a catalog that has the same texts in Spanish (`es`) and English (`en`). The language used SHALL be the `language` setting (a configuration file or `--language`), and it SHALL be `es` when nothing sets it. The language SHALL be fixed for the whole run and SHALL NOT change during a session. Values inside a text (names, numbers, paths, commands) SHALL be the same in both languages.

#### Scenario: Spanish by default
- **WHEN** the user runs `nar` in a folder with no configuration
- **THEN** the banner reads `nar · anthropic · claude-haiku-5-5. Escribe 'exit' para salir.`, with the provider and the model that are configured

#### Scenario: English from the command line
- **WHEN** the user runs `nar --language en`
- **THEN** the banner reads `nar · anthropic · claude-haiku-5-5. Type 'exit' to leave.`

#### Scenario: English from a file
- **WHEN** `.nar/config.toml` sets `language = "en"` and the user runs `nar` without `--language`
- **THEN** every interface text is in English

#### Scenario: Command line wins
- **WHEN** `.nar/config.toml` sets `language = "en"` and the user runs `nar --language es`
- **THEN** every interface text is in Spanish

#### Scenario: Same values in both languages
- **WHEN** the user runs `/model` in each language
- **THEN** the sentences differ, and the provider name, the model name and the source of each value are identical

### Requirement: What is translated
The following SHALL follow the language: the banner and notices of interactive mode, the summary of every command in `/help` and the output of every command, errors and warnings, the prompts (tool approval, trust for `.nar/tools/`, saving an API key, the provider menu), the lines that show tool calls, results and context use, the session, MCP and agent views, and the `--help` text. The prefix `error:` SHALL stay `error:` in Spanish, and the prefix `warning:` SHALL become `aviso:`. A message that is built from several parts SHALL be translated as a whole sentence, not by joining translated words.

#### Scenario: Help summaries
- **WHEN** the user runs `/help` with no configuration
- **THEN** each line shows the command name and its summary in Spanish, for example `/help  lista todos los comandos`, `/exit  sale del modo interactivo` and `/tools  lista las herramientas que el modelo puede llamar`

#### Scenario: Help summaries in English
- **WHEN** the user runs `/help` with `--language en`
- **THEN** the summaries read `list every command`, `leave interactive mode` and `list the tools the model can call`

#### Scenario: Error prefix
- **WHEN** the user runs `nar --resume zzz` and no session matches, with no configuration
- **THEN** the standard error shows `error: no hay ninguna sesión 'zzz'`, and with `--language en` it shows `error: no session 'zzz'`

#### Scenario: Warning prefix
- **WHEN** a skill file is not valid and `nar` starts with no configuration
- **THEN** the warning starts with `aviso:`, and with `--language en` it starts with `warning:`

#### Scenario: Plural messages
- **WHEN** one session file is not valid, and then when two are not valid
- **THEN** each count gets its own correct sentence in the language, for example `1 archivo de sesión no es válido y se omitió` and `2 archivos de sesión no son válidos y se omitieron`

### Requirement: What is never translated
The following SHALL NOT change with the language: command names (such as `/help` and `exit`), option and flag names, configuration keys and their values, file paths, provider and model names, the prompt `nar> `, and text written by hooks, MCP servers or model providers. Text that goes to the model SHALL stay in English whatever the language is: the output and the error messages of tools, the text of a hook refusal, the results of a rejected or unavailable approval, the tool descriptions, and the system, subagent and compaction prompts. A line that is only for the user, such as the one-line summary of a tool result or the title of a preview, SHALL be translated.

#### Scenario: A tool error reaches the model in English
- **WHEN** `edit_file` cannot find the text to replace, in Spanish
- **THEN** the model receives the English error result, and the line the user sees for that call is in Spanish

#### Scenario: Rejection reaches the model in English
- **WHEN** the user answers `n` to a `write_file` call in Spanish
- **THEN** the model receives the same English error result as in English, saying the user rejected the call

#### Scenario: Hook output is untouched
- **WHEN** a hook prints a line in English and the language is Spanish
- **THEN** that line is shown as the hook wrote it

#### Scenario: Command names stay
- **WHEN** the language is Spanish and the user types `/config`
- **THEN** the command runs, and the keys it lists (`provider`, `approval`, `language`) keep their names and values

#### Scenario: An error shown to the user and also to the model
- **WHEN** a message is shown to the user and also reaches the model
- **THEN** the model receives the English text, and the user may see the translated one

### Requirement: Texts quoted in other specs
The user-facing texts quoted in other specs SHALL be the English (`en`) texts. With `language = es`, `nar` SHALL show the Spanish equivalent of each one in the catalog, with the same meaning and the same values.

#### Scenario: A quoted English text in Spanish
- **WHEN** a spec says the user sees `warning: could not compact the conversation: <reason>` and the language is `es`
- **THEN** the user sees the Spanish catalog text that says the same, with the same reason

#### Scenario: Behavior does not depend on the language
- **WHEN** any scenario of another spec is run with `language = es`
- **THEN** the behavior, the exit codes and the values are the same as with `en`, and only the wording of the interface text differs

### Requirement: Prompt answers in both languages
Questions asked to the user SHALL be written in the interface language, and their answers SHALL be accepted in both languages whatever the interface language is. Yes SHALL be `y`, `yes`, `s`, `si` or `sí`; no SHALL be `n` or `no`; always SHALL be `a`, `always` or `siempre`. Letter case and accents SHALL NOT matter. The same answers SHALL apply to the tool approval question, the trust question for `.nar/tools/` and the question about saving an API key. An answer outside the set SHALL be handled as before: the question is asked again, or the answer counts as no where the prompt already says so.

#### Scenario: Approval prompt in Spanish
- **WHEN** the language is Spanish and the model calls `write_file`
- **THEN** the question reads `¿Permitir write_file? [s]í / [n]o / [a] siempre write_file en esta sesión: `

#### Scenario: Approval prompt in English
- **WHEN** the language is English and the model calls `write_file`
- **THEN** the question reads `Allow write_file? [y]es / [n]o / [a]lways allow write_file this session: `

#### Scenario: Trust prompt in Spanish
- **WHEN** the language is Spanish and `.nar/tools/` has files that are not allowed
- **THEN** the question lists the files, says in Spanish that they run as code with the user's permissions, and ends with `¿Ejecutar estos archivos? [s]í / [n]o: `

#### Scenario: API key prompt in Spanish
- **WHEN** the language is Spanish and the user has just typed an API key
- **THEN** the question reads `¿Guardar ANTHROPIC_API_KEY para tu usuario, para la próxima vez? [s]í / [n]o: `

#### Scenario: English answer to a Spanish question
- **WHEN** the language is Spanish and the user answers `YES` to the approval question
- **THEN** the call runs

#### Scenario: Spanish answer without accent
- **WHEN** the language is English and the user answers `SI` to the approval question
- **THEN** the call runs

#### Scenario: Always in both languages
- **WHEN** the user answers `Siempre` in English, or `always` in Spanish
- **THEN** the tool is approved for the rest of the session

#### Scenario: Unknown answer
- **WHEN** the user answers `quizás`
- **THEN** the call does not run and the question is asked again

### Requirement: Output before the configuration is loaded
`--help`, `--version` and the errors for wrong arguments are produced before the configuration is loaded, so `nar` SHALL work out the language first, in this order: the value after `--language` in the arguments; else the `language` in the configuration files, ignoring any error in them; else `es`. These outputs SHALL use that language, including the words that the command-line library adds itself (such as the usage line, the option heading, the help option's description, the error prefix, and the messages for an invalid value, an unknown option or a missing argument). A configuration error that stops the run SHALL be shown in the same pre-resolved language. Option and flag names SHALL NOT change.

#### Scenario: Help in Spanish
- **WHEN** the user runs `nar --help` with no configuration
- **THEN** the usage line starts with `Uso:`, the options heading is `Opciones:`, the option descriptions are in Spanish, and `--help` itself is described as `Muestra este mensaje y sale.`

#### Scenario: Help in English from the option
- **WHEN** the user runs `nar --language en --help`
- **THEN** the usage line starts with `Usage:`, the options heading is `Options:`, and the option descriptions are in English

#### Scenario: Help in English from a file
- **WHEN** `~/.nar/config.toml` sets `language = "en"` and the user runs `nar --help`
- **THEN** the help is in English

#### Scenario: Unknown option
- **WHEN** the user runs `nar --foo` with no configuration
- **THEN** the standard error shows `Error: No existe la opción '--foo'` in Spanish, with `Error:` as the prefix, and `nar` exits with code 2

#### Scenario: Invalid language value
- **WHEN** the user runs `nar --language fr "hello"`
- **THEN** the error is shown in Spanish and lists `es` and `en`, and `nar` exits with code 2

#### Scenario: Broken configuration file
- **WHEN** a configuration file is not valid TOML and the user runs `nar --help`
- **THEN** the help is shown in Spanish (the file is ignored for finding the language), and `nar` exits with code 0

#### Scenario: Configuration error in the pre-resolved language
- **WHEN** `language = "en"` is set in the user file, the project file has an invalid `approval`, and the user runs `nar`
- **THEN** the error that names `approval` is shown in English and `nar` exits with code 1

#### Scenario: Unknown language in a file
- **WHEN** a file sets `language = "fr"` and the user runs `nar --help`
- **THEN** the help is shown in Spanish, and `nar` without `--help` reports the invalid value and exits with code 1

### Requirement: Catalog completeness
The Spanish and the English catalogs SHALL contain exactly the same keys, and for each key both texts SHALL use exactly the same placeholders. A text asked for by a key that does not exist SHALL NOT be shown as an empty line: it SHALL be reported as an internal error that names the key. A test SHALL check the keys and the placeholders of both languages, so that a missing translation is found before release.

#### Scenario: A key is missing in one language
- **WHEN** a key exists in `en` but not in `es`
- **THEN** the catalog test fails and names the key and the language that lacks it

#### Scenario: A placeholder differs
- **WHEN** the `es` text of a key uses `{count}` and the `en` text uses `{n}`
- **THEN** the catalog test fails and names the key

#### Scenario: Tests do not depend on the default language
- **WHEN** the existing test suite runs
- **THEN** the language is fixed to `en` for those tests, and new tests that check Spanish output set `es` explicitly
