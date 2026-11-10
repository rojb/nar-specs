# Spec Delta

## ADDED Requirements

### Requirement: A pasted key is cleaned and refused before it is checked
Before a key pasted at the hidden prompt is used or checked, `nar` SHALL remove surrounding whitespace, one pair of surrounding quotes, and a leading `<ENV>=` where `<ENV>` is the key's variable name. If what remains has a control character or inner whitespace, `nar` SHALL NOT use it or contact the provider. It SHALL print one line that says what is wrong, without showing the key, and ask again. When the control character is Ctrl+V (`\x16`), the line SHALL say to paste with a right-click. An empty answer SHALL still cancel.

#### Scenario: Ctrl+V in the classic console
- **WHEN** the user presses Ctrl+V at the hidden prompt and then Enter
- **THEN** one line says Ctrl+V does not paste here and to paste with a right-click, the provider is not contacted, and the key is asked for again

#### Scenario: Common copy mistakes are fixed
- **WHEN** the user pastes `"sk-abc"` or `DEEPSEEK_API_KEY=sk-abc` for DeepSeek
- **THEN** the key `sk-abc` is the one checked

#### Scenario: Inner spaces
- **WHEN** the pasted text has a space in the middle
- **THEN** one line says a key has no spaces, and the key is asked for again
