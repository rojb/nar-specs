# Tasks

## [specs]

- [x] 0.1 Link this change to Linear RRG-80 (add the `nar-specs` change path as a link) and verify the link opens `openspec/changes/fix-config-utf8-bom/` on GitHub

## [nar]

## 1. Branch

- [x] 1.1 In `nar/`, update `main` and create the branch `rubenjustiniano96/rrg-80-fix-config-utf8-bom-configtoml-con-bom-utf-8-falla-al-cargar`; verify `git status` is clean and `pytest -q` passes before any change

## 2. Loader

- [x] 2.1 In `tests/test_config.py` add tests that fail first: a project file written as `b"\xef\xbb\xbf" + b'approval = "yolo"\n'` loads with `approval == "yolo"` and its provenance is the project file; a user file with a BOM merges like one without; a file with invalid UTF-8 bytes (`b"approval = \"\xff\"\n"`) raises `ConfigError` whose message names the file; existing malformed-TOML tests still report the line. Verify the BOM tests fail on the current code
- [x] 2.2 Change `_read_toml` in `src/nar/config/loader.py` to `tomllib.loads(path.read_bytes().decode("utf-8-sig"))`, keeping the existing `TOMLDecodeError` and `OSError`/`UnicodeDecodeError` handling; verify `pytest tests/test_config.py` passes

## 3. Checks and delivery

- [x] 3.1 Run `ruff check .`, `ruff format --check .` and `pytest -q`; verify all pass
- [x] 3.2 Commit (`fix(config): ...` with `(RRG-80)`), push, open a PR referencing the issue, merge after review

## 4. Manual verification

- [x] 4.1 In a temp folder on Windows PowerShell 5.1, run `New-Item -ItemType Directory .nar; 'approval = "yolo"' | Set-Content -Encoding utf8 .nar\config.toml` (writes a BOM; confirm with `Format-Hex .nar\config.toml` showing `EF BB BF` first), then `nar --show-config` -> prints `approval = yolo  (.nar/config.toml)` and exit 0 (before the fix it printed `invalid TOML ... (line 1)`); write `approval = "always"` the same way -> exit 1 naming `approval` and the allowed values; write the bytes `FF FE` into the file -> exit 1 with `cannot read` naming the file; `nar --version` still prints `nar, version 0.1.0`. Record date, shell version and nar commit
  - Observed on 2026-10-27, Windows 11, Windows PowerShell 5.1.26100.9711, fix code later merged as nar PR #11 (merge 6c14b3d): `Format-Hex` showed `EF BB BF` first; `nar --show-config` printed `approval = yolo  (.nar/config.toml)` and exit 0; `approval = "always"` -> `invalid value "always" for setting 'approval' ... allowed values: ask, auto-edit, yolo`, exit 1; bytes `FF FE` -> `cannot read <file>: 'utf-8' codec can't decode byte 0xff`, exit 1; `nar --version` printed `nar, version 0.1.0`. Before the fix the BOM file gave `invalid TOML ... (line 1): Invalid statement`.
