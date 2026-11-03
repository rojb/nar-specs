# Proposal

- **Owner:** nathaliascode
- **Linear issue:** RRG-70
- **Video timestamps covered:** 14:41:22 (Context Management: Compaction), 15:24:12 (Context Management: Pruning)
- **Depends on:** RRG-63 `add-agent-loop` (merged in `nar`; not archived yet, see Impact)
- **Unblocks:** RRG-76 `add-sessions` and RRG-77 `add-checkpoints` can store and restore a compacted history
- **Course requirement covered:** context management (compaction + pruning)

## Why

The conversation only grows. Every request sends the whole history again, so a long session eventually goes over the model's context window and the provider rejects the call (or, with Ollama, silently drops the oldest text). The token count already exists (RRG-63) but nothing acts on it, and the user cannot see how full the context is.

## What Changes

- **Context limit:** new settings `context_window` (tokens; each provider has a default) and `context_threshold` (percent of the window, default 80). Before every model call `nar` compares the size of the conversation with that limit.
- **Pruning first:** over the limit, the results of the oldest tool calls are replaced by a short marker, keeping the most recent ones. No model call is needed and the history stays valid for the providers.
- **Compaction if pruning is not enough:** `nar` asks the same model for a summary of the older part of the conversation and replaces that part with the summary, keeping the latest messages. The current request is never lost.
- **Manual `/compact`** in interactive mode, with the same compaction.
- **Usage indicator:** in interactive mode, a dim line after each request shows tokens used, the window and the percentage.
- **Failure is not fatal:** when the summary cannot be made, `nar` warns and continues with the history it has. A cancelled or failed request still leaves no trace, including any compaction it did.

Out of scope: retrying automatically when a provider rejects a call as too long, an exact tokenizer per model, looking up the window of a model from the provider, focus instructions for `/compact`, a dedicated cheaper model for summaries, showing the indicator in single mode.

**"Done when" (RRG-70):** a long session keeps working after going over the model's context window.

## Capabilities

### New Capabilities

- `context-compaction`: the context limit, pruning old tool results, compacting with a summary, `/compact`, the usage indicator, failures and cancellation.

### Modified Capabilities

- `configuration`: a new requirement for `context_window` and `context_threshold`, ADDED next to "Initial settings".

## Impact

- **Code (`nar` repo):** new `src/nar/context/pruning.py` and `src/nar/context/compaction.py`; `src/nar/context/manager.py` (snapshots instead of list positions for `mark`/`rollback`, `replace_messages`, tool definition overhead); `src/nar/agent/loop.py` and `src/nar/agent/events.py`; `src/nar/ui/agent_view.py`; `src/nar/cli/runner.py`, `src/nar/cli/repl.py`, `src/nar/cli/slash.py`; `src/nar/config/models.py`; README.
- **Dependencies:** none new.
- **Cost:** a compaction is one extra call to the configured model.
- **Specs:** only a new capability and an ADDED requirement, so this change can be archived before or after `add-agent-loop` and `add-mcp-client`, which still own the `agent-context` and slash command specs.
