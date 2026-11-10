# Proposal

- **Owner:** nathaliascode
- **Linear issue:** RRG-91 (sub-issue of RRG-87)
- **Video timestamps covered:** none (found on 2026-11-10 during the from-scratch test)
- **Depends on:** RRG-87 `add-api-key-prompt` (merged)
- **Course requirement covered:** selectable model provider, as a fix

## Why

In the classic Windows console (conhost), Ctrl+V does not paste into `getpass`'s hidden input: it sends the control character `\x16`. `nar` takes that as the key and checks it, and the provider answers with an error that explains nothing: `deepseek rejected the request (400): Error code: 400`. This happened to Nathalia on 2026-11-10. Other common copy mistakes, such as surrounding quotes or a `DEEPSEEK_API_KEY=` prefix, are reported only as a rejected key.

## What Changes

- Before checking a pasted key, `nar` removes surrounding quotes and a leading `<ENV>=`, which are common copy mistakes.
- A key that still has control characters or inner spaces is not sent to the provider. One line says what is wrong; for Ctrl+V it says to paste with a right-click. Then `nar` asks again.
- An empty answer still cancels.

Out of scope: making Ctrl+V paste in the classic console.

**"Done when" (RRG-91):** pasting with Ctrl+V in the classic console prints a hint about how to paste and asks again, without contacting the provider, and a right-click paste then works.

## Capabilities

### New Capabilities

None.

### Modified Capabilities

- `llm-providers`: a new requirement for cleaning and refusing a pasted key before it is checked. ADDED.

## Impact

- **Code (`nar` repo):** `src/nar/ui/key_prompt.py` (`KeyPrompter.ask_key`), and `tests/test_api_key.py`.
- **Dependencies:** none new.
