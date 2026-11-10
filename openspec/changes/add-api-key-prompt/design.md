# Design

## Context

`factory._api_key` reads the key from `os.environ` and raises `ProviderError` when it is missing. It is called from `create_agent` (startup, both modes) and from `switch_model` (`/model`). `LLMProvider.check()` (RRG-86) can verify a key without a model call. `TerminalPrompter` and `TrustPrompter` already decide whether a person is at the terminal with `_is_terminal(stdin) and _is_terminal(console.file)`.

## Decisions

### 1. `KeyPrompter`
`src/nar/ui/key_prompt.py` (NEW). `available()` uses the same terminal test as the other prompters. `ask_key(env, provider) -> str | None` prints `<ENV> is not set.` and reads with `getpass.getpass`, with the prompt `Paste your <provider> API key (hidden; Enter to cancel): `. An empty answer, `EOFError` or `KeyboardInterrupt` returns `None`. `ask_save(env) -> bool` asks `Save <ENV> for your user, for the next time? [y]es / [n]o: `. An empty or unknown answer asks again, and end of input or Ctrl+C mean no. Both functions take injectable readers, as the other prompters do, so tests drive them without a terminal.

### 2. `obtain_api_key` and `keep_api_key`
`src/nar/cli/api_key.py` (NEW).
- `obtain_api_key(config, prompter) -> PendingKey | None`: when the provider has a key variable (`factory.API_KEY_ENVS`), the variable is empty and the prompter is available, it asks, sets `os.environ[env]` for this process, and returns `PendingKey(env, value)`. Otherwise it returns `None`, and the factory reports the missing key as today.
- `PendingKey.discard()` removes the variable from `os.environ`.
- `keep_api_key(pending, prompter, note)` asks whether to save. On Windows it calls `save_user_variable`, then notes `saved <ENV> for your Windows user; new terminals will see it`. On other systems it notes `not saved: add 'export <ENV>=<your key>' to your shell profile`.
- `save_user_variable(name, value)` writes `winreg.SetValueEx` under `HKEY_CURRENT_USER\Environment`, the same key `setx` writes. Then it broadcasts `WM_SETTINGCHANGE` with `"Environment"` through `SendMessageTimeoutW`, so new terminals see the variable without signing out. A failure is a warning, not an error: the key still works for this session.

**Alternative rejected:** a credentials file in `~/.nar`. It is outside any repository, but it is a plain-text secret on disk and contradicts `Credentials from the environment`.

### 3. Where it is called
- Startup: `src/nar/cli/main.py` (MODIFIED) calls `prepare_api_key(config, prompter, note)` before `run_single` and `repl.run`. This function builds the provider after `obtain_api_key` and runs `check()`. On failure it discards the key and raises; otherwise it calls `keep_api_key`.
- `/model`: `switch_model` (`src/nar/cli/runner.py`, MODIFIED) calls `obtain_api_key` before `create_provider`. If `check()` fails or is cancelled, it discards the pending key. If it passes, it calls `keep_api_key` and then switches.

No new configuration keys.

## Risks / Trade-offs

- **`getpass` on Windows reads from the console, not from `sys.stdin`** → this is why the prompt runs only when both streams are terminals. Pasting works in Windows Terminal and in the classic console.
- **The key stays in the process environment** → shell tools that `nar` runs inherit it, as they do today when the key comes from `setx`.
- **Policy may block the registry write** → it is a warning, and the session still works.

## Migration Plan

None.

## Open Questions

None.
