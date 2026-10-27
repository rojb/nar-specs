# Proposal

- **Owner:** nathaliascode
- **Linear issue:** RRG-67
- **Video timestamps covered:** 10:48:33 (Web Search Tool), 11:03:18 (Web Fetch Tool)
- **Depends on:** RRG-62 `add-tool-framework` (archived)
- **Unblocks:** nothing directly; RRG-73 `add-subagents` can give these tools to its `explorer` subagent
- **Course requirement covered:** none directly

## Why

The agent only knows the project's files and what the model memorized. Coding tasks often need current information that is not in either: a library's documentation, an error message explained online, the API of a newer version. The agent needs to search the web and read a page, without the user configuring an API key.

## What Changes

- Add the built-in **`web_search`** tool: searches the web through DuckDuckGo (no API key) and returns, for each result, its title, URL and snippet, up to a requested number of results (default 5, at most 10).
- Add the built-in **`web_fetch`** tool: downloads an `http`/`https` URL and returns its content for the model: HTML pages converted to Markdown (title, headings, paragraphs, lists, code blocks and absolute links; scripts, styles, navigation and footers removed), plain text, Markdown and JSON returned as text. It has a timeout, a download size limit, and its output is truncated like every tool output.
- Clear error results for network failures, timeouts, HTTP errors, unsupported content (images, PDFs, binaries) and DuckDuckGo refusing the search (rate limiting).
- Neither tool requires approval (they only read).
- Register both as built-ins, so the agent (RRG-63) can use them and `scripts/run_tool.py` can run them.
- New dependencies: `httpx` (HTTP client), `beautifulsoup4` and `markdownify` (HTML parsing and conversion).

Out of scope: running JavaScript (single-page apps are fetched as the server sends them), PDF extraction, logins and cookies, a per-domain permission prompt (RRG-69 may add one), caching.

**"Done when" (RRG-67):** the agent searches for a library's documentation and summarizes a page.

## Capabilities

### New Capabilities

- `web-search-tool`: behavior of the built-in `web_search` tool: query, result fields, result count and errors.
- `web-fetch-tool`: behavior of the built-in `web_fetch` tool: allowed URLs, content conversion, limits and errors.

### Modified Capabilities

<!-- None: the tool contract, the registry export and the agent's behavior do not change at spec level. -->

## Impact

- **Code (`nar` repo):** new `src/nar/tools/web.py` (shared HTTP client settings and error messages), `src/nar/tools/web_search.py`, `src/nar/tools/web_fetch.py`; registration in `src/nar/tools/builtin.py`; README section.
- **Dependencies:** `httpx`, `beautifulsoup4`, `markdownify` added to `pyproject.toml`.
- **Network:** `web_search` sends the query to `html.duckduckgo.com`; `web_fetch` requests the URL the model chooses. Both identify themselves as `nar` in the User-Agent.
- **User-visible:** the agent can look things up online; nothing changes for users who never ask it to.
