# Design

## Context

`KeyPrompter.ask_key(env, provider)` (RRG-87, `src/nar/ui/key_prompt.py`) reads with `getpass` and returns the stripped text, or `None` for an empty answer, end of input or Ctrl+C. In conhost, `getpass` reads characters with `msvcrt.getwch`, so Ctrl+V arrives as `\x16` instead of a paste.

## Decisions

### 1. `clean_key(env, text) -> tuple[str, str | None]`
`src/nar/ui/key_prompt.py` (MODIFIED): a pure function. It returns the cleaned key and, when the key cannot be used, the reason. Steps:
1. Strip surrounding whitespace.
2. Remove one leading `<env>=`.
3. Remove one matching pair of `"` or `'`, and strip again.

The reason is set in these cases:
- `\x16` present → `Ctrl+V does not paste here: paste with a right-click`
- any other character below 32, or 127 → `the key has invisible characters; paste it again`
- inner whitespace → `a key has no spaces; paste it again`

The reason never contains the key.

### 2. `ask_key` asks again
`ask_key` loops:
- an empty answer, end of input or Ctrl+C → returns `None`, as today
- a reason → prints it in yellow and reads again
- otherwise → returns the cleaned key

`obtain_api_key`, `prepare_api_key` and `/model` do not change.

No new configuration keys.

## Risks / Trade-offs

- **A provider might issue keys with characters we refuse** → none of the four supported providers use spaces or control characters in keys.
- **The loop could trap a user who cannot paste** → an empty Enter or Ctrl+C always leaves.

## Migration Plan

None.

## Open Questions

None.
