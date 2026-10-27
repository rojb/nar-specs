# Design

## Context

Tools follow RRG-62's contract (`Tool`, `ToolResult`, JSON Schema validation, 30,000-character output truncation in the registry) and are registered in `src/nar/tools/builtin.py`, now `read_file, write_file, edit_file, list_dir, glob, grep`. The agent (RRG-63) runs tools without approval only when `requires_approval` is false or the level is `yolo`. The `nar` package has no general HTTP client: the provider SDKs bring `httpx2` for their own use. Motivation: see `proposal.md`. Behavior: see `specs/web-search-tool` and `specs/web-fetch-tool`.

Constraints:
- No API keys: the search must work with DuckDuckGo's public HTML page (`https://html.duckduckgo.com/html/`), which answered normally to a `nar/0.1` User-Agent when this design was written.
- Tests run offline.
- Windows and Linux.

## Goals / Non-Goals

**Goals:**
- Readable Markdown for the model from typical documentation pages, with links it can follow.
- Failures always become one-line error results the model can act on.
- No browser automation and no JavaScript.

**Non-Goals:**
- A search provider abstraction (Brave, Tavily, …). DuckDuckGo is the only backend; a second one can be added behind the same tool later.
- PDF text extraction, caching, cookies, robots.txt handling.

## Decisions

### 1. Layout and dependencies
```
src/nar/tools/
├── web.py          NEW  http_client(), USER_AGENT, TIMEOUT_SECONDS, request_error_message()
├── web_search.py   NEW  WebSearchTool, parse_results()
├── web_fetch.py    NEW  WebFetchTool, html_to_markdown()
└── builtin.py      MODIFIED  registers both after GrepTool
pyproject.toml      MODIFIED  + httpx, beautifulsoup4, markdownify
```
- **`httpx`** (sync client, timeouts, redirects, streaming). **Alternative rejected:** `urllib` (manual redirects, charset handling and size limits); reusing the SDKs' `httpx2` (a transitive dependency we do not control).
- **`beautifulsoup4`** parses both DuckDuckGo's results and fetched pages. **`markdownify`** (built on BeautifulSoup) converts the cleaned HTML. **Alternative rejected:** `html2text` (GPL), `readability-lxml` (heavier, C dependency).
- **Alternative rejected for search:** the `ddgs` package — a meta-search library with frequent breaking changes and its own scraping; parsing one stable HTML page ourselves keeps the behavior testable with small fixtures.

### 2. Shared HTTP settings (`web.py`)
`http_client()` returns `httpx.Client(timeout=20.0, follow_redirects=True, max_redirects=5, headers={"User-Agent": "nar/<version> (+https://github.com/rojb/nar)"})`, used as a context manager per call. `request_error_message(url, exc)` maps `httpx.TimeoutException` → `request to <url> timed out after 20 s`, `httpx.TooManyRedirects` → `too many redirects for <url>`, other `httpx.HTTPError` (DNS, refused, TLS) → `could not reach <url>: <reason>`.

### 3. `web_search`
Schema: `query` (string, `minLength: 1`, required), `max_results` (integer, 1–10, optional, default 5); `additionalProperties: false`; `requires_approval = False`.
- `POST https://html.duckduckgo.com/html/` with form `q=<query>`.
- Status 202, 403 or 429, or a page whose form/markup mentions `anomaly`/`challenge` without results → error `DuckDuckGo refused the search (probably rate limiting); try again later`. Other non-200 statuses → `DuckDuckGo answered HTTP <code>`. Network errors → `could not reach DuckDuckGo: <reason>`.
- `parse_results(html)`: each `div.result` that is not `.result--ad`; title and link from `a.result__a`, snippet from `.result__snippet` (whitespace collapsed). A link of the form `//duckduckgo.com/l/?uddg=<encoded>` is decoded to the destination URL; protocol-relative links get `https:`.
- Output, one block per result:
  ```
  1. <title>
     <url>
     <snippet>
  ```
  No results → `(no results for <query>)`. `summary = "N results"`.

### 4. `web_fetch`
Schema: `url` (string, required); `additionalProperties: false`; `requires_approval = False`.
- Scheme check before any I/O: only `http`/`https` → otherwise `only http and https URLs are supported: <url>`.
- `client.stream("GET", url)`; status ≥ 400 → `HTTP <code> for <url>`. Read chunks up to 5 MB (`MAX_DOWNLOAD_BYTES`); beyond that the body is cut and the output says so (`[download stopped at 5 MB]`).
- Content type from the header (`text/html` or `application/xhtml+xml` → HTML; `text/*`, `application/json`, `application/xml`, `+json`, `+xml` → text; missing header → sniff `<html` in the first bytes, else text if it decodes as UTF-8) → otherwise `content type <type> is not supported` (images, PDFs, archives).
- Decoding: `response.charset_encoding` or UTF-8, `errors="replace"`.
- `html_to_markdown(html, base_url) -> (title, markdown)`: BeautifulSoup (`html.parser`); title from `<title>`; remove `script, style, noscript, nav, footer, aside, form, iframe, svg, template` and `<head>`; links (`a[href]`) and images (`img[src]`) made absolute with `urljoin(final_url, ...)`; `markdownify(..., heading_style="ATX", bullets="-")`; runs of 3+ blank lines collapsed to one.
- Output: `# <title>` (when present), `URL: <final url>`, blank line, content. Length is capped by the registry (30,000 characters with its marker); no second limit here. `summary = "fetched <n> chars from <host>"`.

### 5. Approval and safety
Both tools only read and are `requires_approval = False`, so the agent can use them at the default `ask` level, as the issue's "done when" expects. Trade-off: a prompt-injected page could make the model fetch a URL that encodes data in its query string. Mitigations now: the tool activity line shows every URL to the user; RRG-69 can turn on approval for `web_fetch` (or per domain) by changing one flag. Local-network addresses are not blocked (developers fetch `localhost` docs servers).

### 6. Tests
Offline: `httpx.MockTransport` injected through a module-level factory that the tests patch (`nar.tools.web.http_client`). `tests/test_tools_web_search.py` uses small hand-written HTML in DuckDuckGo's markup (normal results, an ad, a redirect link, empty page, challenge page, 202 status, connection error). `tests/test_tools_web_fetch.py` covers HTML conversion (nav/script removed, absolute links, code block, title), redirects, JSON, PDF rejection, 404, timeout, unknown host, the scheme check, the 5 MB cut and an 80,000-character page truncated by the registry. Live requests happen only in the manual checklist.

## Risks / Trade-offs

- [DuckDuckGo changes its HTML or rate-limits] → Clear error results; parsing is isolated in `parse_results()` with fixture tests, so a markup change is a one-function fix.
- [JavaScript-only sites return almost nothing] → The model sees a short page and can try another URL; out of scope here.
- [Prompt injection from fetched pages] → See Decision 5.
- [Three new dependencies] → All widely used and pure Python; `pip install -e ".[dev]"` installs them.

## Migration Plan

Run `pip install -e ".[dev]"` again for the new dependencies. No configuration or data changes. Rollback: revert the PR.
