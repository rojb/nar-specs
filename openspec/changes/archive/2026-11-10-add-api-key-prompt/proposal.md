# Proposal

- **Owner:** nathaliascode
- **Linear issue:** RRG-87 (sub-issue of RRG-61)
- **Video timestamps covered:** none (asked for by Nathalia on 2026-11-10, while trying `nar` as a new user)
- **Depends on:** RRG-61 `add-llm-providers` and RRG-86 `fix-model-switch-check` (both merged)
- **Course requirement covered:** selectable model provider

## Why

When the provider's key is missing, `nar` stops with `error: DEEPSEEK_API_KEY is not set`, and the user has to know what an environment variable is and how to create one. Other CLIs, such as Claude Code or `gh`, ask for the credential the first time.

## What Changes

- When the key is missing and a person is at the terminal, `nar` asks for it with hidden input, at startup and in `/model`.
- The pasted key is checked with `LLMProvider.check()` before it is used. A rejected key is neither used nor saved.
- After the check, `nar` asks whether to save it. On Windows it is saved as a user environment variable, like `setx`, never in a file, so it cannot end up in a repository. On other systems it is not saved, and `nar` shows the `export` line to add to the shell profile, without the key.
- The key is never printed.
- Without a terminal, nothing changes: the same one-line error.

Out of scope: replacing a key that is set but invalid, and saving keys in files.

**"Done when" (RRG-87):** in a new terminal without `DEEPSEEK_API_KEY`, `nar --provider deepseek` asks for the key without showing it, checks it, saves it when the answer is yes, and in another new terminal `nar --provider deepseek` starts without asking.

## Capabilities

### New Capabilities

None.

### Modified Capabilities

- `llm-providers`: `Credentials from the environment` is MODIFIED, so a missing key is asked for when a terminal is attached, and is still an error otherwise. A new requirement covers asking for and saving the key. ADDED.

## Impact

- **Code (`nar` repo):** `src/nar/ui/key_prompt.py` (new), `src/nar/cli/api_key.py` (new), `src/nar/cli/runner.py`, and tests.
- **Dependencies:** none new. `getpass` and `winreg` are in the standard library.
- **Security:** the key lives only in the process environment and, if the user agrees, in the Windows user environment. It is never written to a file, a session or a message.
