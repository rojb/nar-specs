# Design

## Context

When a request fails, `Agent.run` rolls it back: the conversation is as it was before the user's line, so running the same line again is safe. `repl.run` catches `ProviderError` from `run_request` and prints it. `switch_model` (RRG-86, RRG-87) already asks for a missing key, checks the provider, and switches the agent and the compactor. `translate_error` and `_exhausted` in `src/nar/providers/retry.py` build every provider error.

## Decisions

### 1. Marking the errors
`src/nar/errors.py` (MODIFIED): `ProviderError` gets a keyword-only `unavailable: bool = False`.

`src/nar/providers/retry.py` (MODIFIED) sets it in these cases:
- in `translate_error`: connection errors, HTTP 402, and HTTP 400 whose message contains "credit balance"
- in `_exhausted`: always, since it is only reached after retryable failures (timeouts, connection errors, 429, 529 and 5xx)
- in `call_once`, for a timeout

No other error is marked.

### 2. `ProviderMenu`
`src/nar/ui/provider_menu.py` (NEW). `available()` uses the same terminal test as the other prompters. `choose(failed, options) -> str | None` does the following:
- It prints `<failed> can't answer. Continue with another provider?`.
- It prints one numbered line per option: name, default model, and `key set`, `no key: it will ask for one` or `local, no key`.
- It prints `Enter: stay on <failed>`, then reads `Choose 1-N: `.

An empty answer, end of input or Ctrl+C returns `None`, and an unknown answer asks again. The reader can be injected, as in the other prompters.

### 3. `offer_other_provider`
`src/nar/cli/fallback.py` (NEW): `retry_elsewhere(agent, view, line, recorder, error, menu) -> bool`.
- It returns `False` at once when `error.unavailable` is false or the menu is not available.
- Otherwise it loops:
  - It builds the options from `ProviderName`, without the providers tried so far, using `DEFAULT_MODELS` and `factory.API_KEY_ENVS`.
  - It asks the menu, and stops on `None`.
  - It calls `switch_model(agent, view, name)`. On `NarError` it prints the error and asks again; on Ctrl+C it stops.
  - It runs `run_request(agent, view, line, recorder)` and returns `True`.
  - If that also raises a `ProviderError` marked as unavailable, it prints the error and loops with that provider added to the ones tried. Any other error propagates, as today.

`src/nar/cli/repl.py` (MODIFIED): in the `except ProviderError` branch, after `view.error`, it calls `retry_elsewhere`.

No new configuration keys.

## Risks / Trade-offs

- **Tool calls that ran before the failure are not undone on disk** → this is today's rollback behavior: the retried request may repeat them. Edits are usually idempotent, and approval still applies.
- **Retries before the list** → a 429 or a server error still goes through the four attempts (about 8 seconds) before the list appears, so a short outage is not mistaken for a down provider.
- **Different prices** → the user picks; nothing switches without asking.

## Migration Plan

None.

## Open Questions

None.
