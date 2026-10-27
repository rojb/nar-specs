# Design

## Context

`src/nar/config/loader.py` reads each configuration file in `_read_toml(path)` by opening it in binary mode and calling `tomllib.load(handle)`. `tomllib` decodes the bytes as strict UTF-8 and treats a leading BOM (`EF BB BF`, decoded as `U+FEFF`) as an invalid character, so it raises `TOMLDecodeError("Invalid statement (at line 1, column 1)")`, which the loader reports as `invalid TOML in <path> (line 1)`. Motivation: see `proposal.md`. Required behavior: see the `configuration` delta.

## Goals / Non-Goals

**Goals:** a configuration file with a UTF-8 BOM loads exactly like the same file without it; existing errors (malformed TOML with file and line, unreadable file) keep their wording.

**Non-Goals:** other encodings (UTF-16, cp1252) are not supported; `AGENTS.md` and tool file reads are out of scope.

## Decisions

### Decision 1: Decode with `utf-8-sig`, then `tomllib.loads`

`_read_toml` reads the bytes, decodes them with the `utf-8-sig` codec (strips one leading BOM, otherwise identical to `utf-8`) and parses the text with `tomllib.loads`. A `UnicodeDecodeError` is already caught and reported as `cannot read <path>: ...`, so invalid UTF-8 keeps a clear error naming the file.

Alternative considered: strip the three BOM bytes before `tomllib.load` on a `BytesIO`. Rejected: same effect with more code; the codec is the standard way to express "UTF-8 with optional BOM".

## Modules touched

- `nar/config/loader.py`, function `_read_toml`: **modified**.
- `tests/test_config.py`: **modified** (new tests).

## Config keys

None.

## Risks / Trade-offs

- A file whose first character is intentionally `U+FEFF` would lose it: not meaningful for TOML, where that character is invalid at the start anyway.
