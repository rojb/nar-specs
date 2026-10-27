# Tasks

## [specs]

- [x] 0.1 Link this change to Linear RRG-67 (move the issue to In Progress and add the `nar-specs` change path as a link) and verify the link opens `openspec/changes/add-web-tools/` on GitHub once merged
- [x] 0.2 Open a PR in `nar-specs` from Linear's RRG-67 branch with proposal, specs, design and tasks; verify `openspec validate add-web-tools --strict` passes

## [nar]

## 1. Branch and dependencies

- [x] 1.1 In `nar/`, update `main` and create the branch `nathaliascode/rrg-67-add-web-tools-web_search-y-web_fetch`; verify `git status` shows the new branch and a clean tree
- [x] 1.2 Add `httpx`, `beautifulsoup4` and `markdownify` to `pyproject.toml`; verify `pip install -e ".[dev]"` succeeds and `python -c "import httpx, bs4, markdownify"` works

## 2. Shared web helpers

- [x] 2.1 Create `src/nar/tools/web.py` (`USER_AGENT`, `TIMEOUT_SECONDS = 20`, `http_client()`, `request_error_message()`, design.md §2) with `tests/test_tools_web.py`; verify the client sends the `nar/<version>` User-Agent, follows redirects, and that timeout, too-many-redirects and connection errors give the three messages

## 3. web_search tool

- [x] 3.1 Create `src/nar/tools/web_search.py` (`WebSearchTool`, `parse_results`, design.md §3) with `tests/test_tools_web_search.py` on hand-written DuckDuckGo-style HTML through `httpx.MockTransport`; verify every scenario of `specs/web-search-tool/spec.md`: numbered results with title, URL and snippet, ads skipped, `uddg` redirect links decoded, default of 5, `max_results` 50 rejected by the registry, no results, offline, challenge page and HTTP 202 refused with the try-again message; plus the request is a POST with `q=<query>` and `WebSearchTool().requires_approval is False`

## 4. web_fetch tool

- [x] 4.1 Create `src/nar/tools/web_fetch.py` (`WebFetchTool`, `html_to_markdown`, design.md §4) with `tests/test_tools_web_fetch.py` through `httpx.MockTransport`; verify every scenario of `specs/web-fetch-tool/spec.md`: title + final URL + Markdown, redirect followed with the final URL shown, `file://` and `ftp://` rejected without a request, nav/script/footer removed with `# Install`, an absolute link to `/guide` and the code block kept, JSON returned as text, PDF rejected naming `application/pdf`, an 80,000-character page truncated by the registry marker, timeout, 404 with the URL, unknown host; plus a body over 5 MB is cut with `[download stopped at 5 MB]`, a missing content type with HTML is converted, and `WebFetchTool().requires_approval is False`

## 5. Registration and docs

- [x] 5.1 Register `WebSearchTool` and `WebFetchTool` after `GrepTool` in `src/nar/tools/builtin.py` and update `tests/test_tools_registry.py`; verify `create_default_registry().names()` ends with `"glob", "grep", "web_search", "web_fetch"` and both definitions export only `name`, `description`, `parameters`
- [x] 5.2 Add the two tools to the README's built-in tools table and a short paragraph (DuckDuckGo without a key, rate limiting, what `web_fetch` returns and its limits, no approval) with `scripts/run_tool.py` examples; verify the documented commands run as written from the repo root

## 6. Delivery

- [x] 6.1 Run `ruff check .`, `ruff format --check .` and `pytest`; verify all three pass, then push the branch and open a PR in `nar` referencing RRG-67

## 7. Manual verification (RRG-67 "done when")

- [ ] 7.1 Walk through RRG-67's "done when" ("the agent searches for a library's documentation and summarizes a page") at the `nar/` repo root with `NAR_HOME` pointing at a temp folder, and record what was observed for each step:
  - `python scripts/run_tool.py web_search '{"query": "httpx python documentation", "max_results": 3}'` → three numbered results with title, URL and snippet
  - `python scripts/run_tool.py web_fetch '{"url": "https://www.python-httpx.org/"}'` → title, URL and readable Markdown with absolute links, no navigation menu
  - `nar --provider openrouter "busca la documentación de la librería httpx y resume su página principal"` → stderr shows `web_search` then `web_fetch`; the answer summarizes the page; exit 0
  - Same prompt with `--provider ollama --model <local model>` (and `anthropic` when a key is available)

  Observed on 2026-10-27 (Linux, `NAR_HOME` in a temp folder, nar branch `nathaliascode/rrg-67-add-web-tools-web_search-y-web_fetch`):
  - `web_search` "httpx python documentation" (first live run) → `0 results`: DuckDuckGo answered its "No results found" page for a query that has results; repeated requests showed this on about 1 in 3 requests in bursts. Fixed with up to 2 retries (design.md §3); afterwards 6/8 distinct queries answered on the first call and 10/10 with spaced requests ✔
  - `web_fetch` `https://www.python-httpx.org/` → `# HTTPX`, `URL: https://www.python-httpx.org/`, clean Markdown (install command, example session, features) with absolute links, no menu; 4,193 characters ✔. A `<title>` outside `<head>` was duplicated in the body; fixed and covered by a test ✔
  - OpenRouter (`openrouter/auto`): `nar --provider openrouter "busca la documentación de la librería httpx y resume su página principal"` → stderr `web_search` ×2 (✓ 5 results each), `web_fetch(url="https://www.python-httpx.org/")` (✓ fetched 4193 chars); the answer summarized what HTTPX is, installation, main features and linked the QuickStart/Async/HTTP2/API pages; 15.3 s, exit 0 ✔
  - Ollama (`qwen3.5:4b`, CPU): running; Anthropic: **pending** (no `ANTHROPIC_API_KEY`)
