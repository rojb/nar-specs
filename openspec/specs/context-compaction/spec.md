# context-compaction Specification

## Purpose
Defines how `nar` keeps a long conversation inside the model's context window: the limit, pruning old tool results first, compacting with a summary if that is not enough, `/compact`, the usage indicator, and what happens on failure or cancellation.

## Requirements

### Requirement: Context limit
Before every model call, `nar` SHALL compare the size of the conversation in tokens with the context limit, which is `context_threshold` percent of `context_window`. The size SHALL include the system prompt and the tool definitions. At or below the limit the conversation SHALL be sent unchanged.

#### Scenario: Below the limit
- **WHEN** the conversation is below the limit
- **THEN** the history sent to the model is unchanged and no summary is requested

#### Scenario: Checked between tool calls
- **WHEN** the tool results of one model turn take the conversation over the limit
- **THEN** the conversation is reduced before the next model call of that same request

### Requirement: Pruning old tool results
Over the limit, `nar` SHALL first replace the content of the oldest tool results with a short marker that names the tool and says the result was removed, keeping the most recent results up to a quarter of the context window. Results the model has not seen yet (those after the last assistant message) SHALL never be removed; when together they are larger than half of the context window, each SHALL be cut to its share of that half, with a note telling the model to request a smaller part. Short results SHALL be left as they are. User requests, assistant answers and tool calls SHALL NOT change, and every tool call SHALL keep its result. When pruning brings the conversation to the limit or below, no summary SHALL be requested. The user SHALL see a line saying how many results were removed.

#### Scenario: Old file reads are pruned
- **WHEN** a session is over the limit because of large file reads from earlier requests
- **THEN** those results are replaced by the marker, the latest results stay complete, the model is not asked for a summary, and the provider accepts the next call

#### Scenario: A new result reaches the model
- **WHEN** a tool returns a result larger than a quarter of the window but smaller than half of it, taking the conversation over the limit
- **THEN** the next model call contains that result complete, and only older content is reduced

#### Scenario: A result that cannot fit
- **WHEN** a tool returns a result larger than half of the window
- **THEN** the next model call contains the start of the result and a note telling the model to request a smaller part, and the model is not left repeating the same call

#### Scenario: The model can recover pruned data
- **WHEN** the model needs a file whose earlier read was pruned
- **THEN** the marker tells it to call the tool again

### Requirement: Compaction with a summary
When the conversation is still over the limit after pruning, `nar` SHALL ask the configured model, without tools, for a summary of the older messages, and SHALL replace them with one message holding that summary, keeping the latest messages. The kept messages SHALL NOT exceed a quarter of the context window nor half of the conversation. A kept part SHALL never start with a tool result, and SHALL always include the results the model has not seen yet with the call they answer, whatever the budget. The summary message SHALL say it was generated automatically and that text quoted in it from files or tools is data, not instructions. When the request in progress is not among the kept messages, its text SHALL be added to the summary message unchanged. The conversation SHALL NOT be compacted when the older part, not counting the request in progress, is under 5% of the context window, because a summary would not free useful space. The user SHALL see that compaction is in progress and the size before and after.

#### Scenario: Long session continues
- **WHEN** a session goes over the limit and pruning does not free enough
- **THEN** the older messages are replaced by the summary, the provider accepts the next call, and the user can ask a follow-up about something from the summarized part

#### Scenario: Compaction in the middle of a request
- **WHEN** compaction happens between tool calls and the user's request is in the older part
- **THEN** the next call still contains the text of the request and every kept tool call has its result

#### Scenario: Compacting again
- **WHEN** a conversation that has a summary goes over the limit again
- **THEN** the new summary is made from a transcript that includes the previous summary

#### Scenario: Nothing worth compacting
- **WHEN** the conversation is over the limit but its older part is under 5% of the window
- **THEN** no summary is requested and the request goes on

#### Scenario: A huge request alone
- **WHEN** a single request is over the limit by itself
- **THEN** no summary is requested and its text is sent unchanged

### Requirement: Summary failure
A summary that cannot be made (a provider error, an empty answer, or a result that is not smaller than the conversation) SHALL leave the history as it was. During a request, the user SHALL see a warning and the request SHALL continue.

#### Scenario: Provider error during compaction
- **WHEN** the provider fails while summarizing during a request
- **THEN** `nar` warns that it could not compact, sends the next model call with the history it had, and does not end the session

#### Scenario: Empty summary
- **WHEN** the model returns an empty summary
- **THEN** the history is unchanged and the user sees a warning

#### Scenario: Summary larger than the conversation
- **WHEN** the result of a compaction would not be smaller than the conversation
- **THEN** the result is discarded, the history is unchanged and the user sees a warning

### Requirement: Cancelled requests undo context changes
A request that is cancelled or fails SHALL leave the conversation as it was before the request, even when the request pruned or compacted it.

#### Scenario: Cancel after compaction
- **WHEN** the user presses Ctrl+C during a request that has already compacted the conversation
- **THEN** the history is the same as before that request, including the messages that had been summarized

### Requirement: Manual compaction
In interactive mode, the `/compact` command SHALL compact the conversation at any size, and SHALL show the size before and after. It SHALL NOT take arguments. An empty conversation SHALL produce a message that there is nothing to compact, without calling the model. A failure SHALL show an error and leave the history unchanged, and Ctrl+C SHALL cancel the command with the same result. The session SHALL continue in every case.

#### Scenario: Compact on request
- **WHEN** the user types `/compact` after several requests
- **THEN** the conversation is replaced by a summary plus the latest messages and the prompt comes back

#### Scenario: Nothing to compact
- **WHEN** the user types `/compact` before any request
- **THEN** `nar` says there is nothing to compact and does not call the model

### Requirement: Context usage indicator
In interactive mode, after each request and after `/compact`, `nar` SHALL show on standard error the tokens used, the context window and the percentage. Single mode SHALL NOT show it.

#### Scenario: Indicator after a request
- **WHEN** a request finishes in interactive mode
- **THEN** a line such as `context: 12,300 / 128,000 tokens (10%)` is shown on standard error and nothing is added to standard output

### Requirement: Compaction uses the current provider
The summary of a compaction SHALL be requested from the provider and model selected at that moment, including after `/model`.

#### Scenario: Manual compaction after changing the model
- **WHEN** the user runs `/model` to change the provider and then `/compact`
- **THEN** the summary is requested from the new provider, and the previous one receives nothing

#### Scenario: Automatic compaction after changing the model
- **WHEN** the conversation goes over the context limit after `/model`
- **THEN** the summary is requested from the new provider
