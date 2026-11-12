# tui-commands Specification

## Purpose
Defines the interactive command surface of `nar`: the router that runs a `/` command, the completion
offered at the prompt, the commands every session has, and what the commands that change the session
in place do to the conversation.

## Requirements

### Requirement: Slash commands are listed and completed

In interactive mode, typed text starting with `/` SHALL be a command and SHALL be routed to the command registered under that name. Entering `/help` SHALL list every registered command as `/name` followed by its one-line summary, in alphabetical order. When standard input is a terminal, the prompt SHALL complete command names as the user types a `/` command and SHALL show the summary of the highlighted one. When standard input is not a terminal (a pipe or a redirected file), the prompt SHALL read a plain line without completion and everything else SHALL behave the same.

#### Scenario: Help lists everything registered
- **WHEN** the user types `/help`
- **THEN** every registered command appears with its summary, including the ones other changes added (`/mcp`, `/compact`, `/sessions`, `/resume`)

#### Scenario: Completion
- **WHEN** the user types `/co` at a terminal prompt
- **THEN** `/compact` and `/config` are offered with their summaries

#### Scenario: A line that is not a command
- **WHEN** the user types `hello /help` or `/etc/hosts is what`
- **THEN** the line is sent to the model as a request, not treated as a command

#### Scenario: Piped input
- **WHEN** the input is piped (`printf '/help\n' | nar`)
- **THEN** the command runs and no completion machinery is loaded

#### Scenario: An unknown command
- **WHEN** the user types `/nope`
- **THEN** an error lists the registered commands and the session continues

### Requirement: The base commands

Every session SHALL have `/help`, `/clear`, `/exit`, `/model`, `/approval`, `/config`, `/tools` and `/tokens`, in addition to the commands other changes register. `exit` without a slash SHALL keep working as it does today. No command SHALL write its output to standard output: it goes to standard error as literal text, so a redirected run keeps only the answer.

#### Scenario: Exit
- **WHEN** the user types `/exit`
- **THEN** the session ends with exit code 0, exactly like `exit`

#### Scenario: A command that takes no argument
- **WHEN** the user types `/help extra`
- **THEN** the command runs and the argument is ignored, except where a command documents one

#### Scenario: A command registered by another change answers on standard error
- **WHEN** the user types `/mcp`, `/sessions` or `/checkpoints`
- **THEN** the answer appears on standard error and standard output holds only the banner and the prompt

#### Scenario: A redirected interactive run keeps only the answers
- **WHEN** `nar` reads commands from standard input and standard output is redirected to a file
- **THEN** the file holds only the banner, the prompt and the model's answers, never a command's answer

### Requirement: /model changes provider and model in place

`/model` with no argument SHALL show the provider and model in use, with the source they came from. `/model <provider>` SHALL switch to that provider's default model, `/model <provider>/<model>` and `/model <provider> <model>` SHALL switch to both, and an unknown provider SHALL be rejected with the list of providers without changing anything. A switch SHALL keep the conversation, the tools and the session, SHALL apply to the next model call, and SHALL report the new provider and model. A switch to a provider whose credentials are missing SHALL fail with a one-line error, and the previous provider SHALL stay in use.

#### Scenario: Switching providers keeps the conversation
- **WHEN** the user asks a question, types `/model ollama`, and asks a follow-up
- **THEN** the session stays, the follow-up is answered by Ollama, and it can refer to the earlier answer

#### Scenario: Missing credentials
- **WHEN** the user types `/model anthropic` without an Anthropic key
- **THEN** a one-line error says the variable is missing and the next request still uses the previous provider

#### Scenario: Showing the current one
- **WHEN** the user types `/model`
- **THEN** the provider, the model and the source are shown and nothing changes

### Requirement: /approval changes the autonomy level in place

`/approval` with no argument SHALL show the level in use. `/approval ask|auto-edit|yolo` SHALL change it for the following tool calls in the same session, and an unknown level SHALL be rejected with the three names without changing anything.

#### Scenario: From ask to yolo
- **WHEN** the user types `/approval yolo` and then asks for a shell command
- **THEN** the command runs without asking

### Requirement: /clear starts a new conversation

`/clear` SHALL replace the conversation with the system prompt alone, SHALL say that the earlier messages are gone, and SHALL keep the session, the provider and the approval level. Later requests SHALL NOT contain any earlier message. The session file SHALL NOT keep the discarded messages.

#### Scenario: Clearing
- **WHEN** the user asks something, types `/clear`, and asks again
- **THEN** the second request is sent with no earlier message and the session continues

### Requirement: Inspecting the session

`/config` SHALL show the effective configuration with the file each value came from, hiding secrets the way `--show-config` does. `/tools` SHALL list every registered tool name with its one-line description, including the MCP tools of the connected servers. `/tokens` SHALL show the tokens used, the context window and the percentage, like the line shown after each request.

#### Scenario: Secrets stay hidden
- **WHEN** a server header holds a literal secret and the user types `/config`
- **THEN** the value is shown masked

#### Scenario: MCP tools are listed
- **WHEN** an MCP server is connected and the user types `/tools`
- **THEN** its `mcp__<server>__<tool>` names appear with the built-in tools

### Requirement: /model checks the new provider before switching
Before switching, `/model` SHALL check the new provider and model. While it checks, it SHALL say so on standard error. When the check fails, `/model` SHALL print one error line that ends by naming the provider and model still in use, and the agent, compaction and the next request SHALL keep the previous provider. Ctrl+C during the check SHALL cancel the switch with the same guarantee, and the session SHALL continue.

#### Scenario: Ollama is stopped
- **WHEN** the user types `/model ollama` and Ollama is not running, then asks a question
- **THEN** one error line says Ollama cannot be reached and that the previous provider is still in use, and the question is answered by the previous provider

#### Scenario: An invalid key
- **WHEN** the user types `/model deepseek` with a `DEEPSEEK_API_KEY` that DeepSeek rejects
- **THEN** one error line names `DEEPSEEK_API_KEY`, and nothing changes

#### Scenario: A misspelled model
- **WHEN** the user types `/model anthropic claude-sonet-5-5`
- **THEN** the error says the model was not found, suggests close names, and nothing changes

#### Scenario: Cancelling the check
- **WHEN** the user presses Ctrl+C while `/model` is checking
- **THEN** the switch is cancelled, the previous provider stays and the prompt comes back

#### Scenario: A successful switch
- **WHEN** the check passes
- **THEN** `/model` switches and reports the new provider and model, as before

### Requirement: /models lists a provider's models
`/models` SHALL list the models of the provider in use, sorted by name, one per line on standard error, and SHALL mark the model in use. `/models <provider>` SHALL list that provider's models without switching, and an unknown provider SHALL be rejected with the list of providers. `/models <provider> <text>` SHALL keep only the names that contain the text, ignoring case, and SHALL say so when none match. A missing key, a rejected key or a server that does not answer SHALL print one error line, and the session SHALL continue. Ctrl+C while listing SHALL cancel the listing only.

#### Scenario: The provider in use
- **WHEN** DeepSeek is in use with `deepseek-flash`, and the user types `/models`
- **THEN** DeepSeek's models are listed, `deepseek-flash` is marked as in use, and the provider stays the same

#### Scenario: Another provider
- **WHEN** the user types `/models anthropic` with `ANTHROPIC_API_KEY` set
- **THEN** Anthropic's models are listed, and the provider in use does not change

#### Scenario: Filtering
- **WHEN** the user types `/models openrouter claude`
- **THEN** only OpenRouter's models whose names contain `claude` are listed, with how many there are

#### Scenario: No match
- **WHEN** the filter matches no name
- **THEN** one line says that no model matches, and how many models the provider has

#### Scenario: A missing key
- **WHEN** the user types `/models anthropic` without `ANTHROPIC_API_KEY`
- **THEN** one error line names `ANTHROPIC_API_KEY`, and the session continues

### Requirement: Another provider is offered when the current one cannot answer
In interactive mode, with both standard input and standard error at a terminal, when a request fails with an error marked as "cannot answer", `nar` SHALL print the error and then list every other provider on standard error. Each provider SHALL appear with a number, its default model, and whether its key is set or will be asked for. The user SHALL be able to pick a number, or press Enter to stay.

A pick SHALL switch as `/model <provider>` does: the key is asked for when missing, and the provider is checked. When the switch fails, its error is printed and the list is shown again. After a switch, the same request SHALL run again on the new provider, with the conversation as it was before the failed request. When that request also fails with a "cannot answer" error, the list SHALL be offered again, without the providers already tried.

Enter, end of input or Ctrl+C at the list SHALL keep the previous provider, as before this change. No list SHALL be shown for errors that are not marked, in single mode, or without a terminal.

#### Scenario: Picking another provider
- **WHEN** DeepSeek has no balance, the user asks a question and picks `anthropic` with its key set
- **THEN** the error and the list are shown, Anthropic is checked, the switch is reported, and Anthropic answers the same question

#### Scenario: Staying
- **WHEN** the list is shown and the user presses Enter
- **THEN** nothing switches, the request stays failed, and the prompt comes back

#### Scenario: A pick that fails its check
- **WHEN** the user picks `ollama` and Ollama is not running
- **THEN** the check's error is printed and the list is shown again

#### Scenario: Not offered
- **WHEN** a request fails because the key is rejected, or `nar` runs in single mode or with piped input
- **THEN** only the error is printed, as before

### Requirement: /skills lists the available skills

`/skills` SHALL list every available skill on standard error, in name order, one per line, with its name, its description, and whether it came from the project or the user. When no skill is available, `/skills` SHALL say so and name the two folders where skills are looked for.

#### Scenario: Listing
- **WHEN** the project skill `demo` and the user skill `release` are available and the user types `/skills`
- **THEN** both appear with their descriptions, `demo` marked as project and `release` marked as user

#### Scenario: Nothing found
- **WHEN** no skill is available and the user types `/skills`
- **THEN** a line says no skills were found and names `.nar/skills/` and the user skills folder

### Requirement: /skill uses a skill from the prompt

`/skill <name>` SHALL add the instructions of that skill to the next request sent to the model, marked as the skill the user chose, and SHALL say on standard error that the skill will be used. `/skill <name> <request>` SHALL send `<request>` right away, with the skill's instructions added in the same way. The instructions SHALL be read from disk when the command runs. `/skill` with no name, or with a name that is not available, SHALL print an error that lists the available skills, and nothing SHALL be added to the conversation.

#### Scenario: Skill for the next request
- **WHEN** the user types `/skill demo` and then `write the changelog`
- **THEN** the request sent to the model holds the instructions of `demo` and the text `write the changelog`

#### Scenario: Skill and request together
- **WHEN** the user types `/skill demo write the changelog`
- **THEN** the request is sent at once, with the instructions of `demo` and the text `write the changelog`

#### Scenario: Unknown skill
- **WHEN** the user types `/skill nope`
- **THEN** an error lists the available skills and the next request is sent without any skill

#### Scenario: Cleared before use
- **WHEN** the user types `/skill demo` and then `/clear`
- **THEN** the next request is sent without the instructions of `demo`
