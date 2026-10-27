# Spec Delta

## Purpose

Defines the built-in `web_search` tool that lets the agent search the web without an API key, what each result contains, how many results it returns, and the errors it reports.

## ADDED Requirements

### Requirement: Search the web
`web_search` SHALL accept a required `query` and return web search results from DuckDuckGo, without any API key. Each result SHALL show its title, its URL and a short snippet of text, numbered in the order the search engine returned them. Advertisements SHALL NOT be included. It SHALL NOT require user approval.

#### Scenario: Documentation search
- **WHEN** `web_search` is called with `query` "httpx python documentation"
- **THEN** the result is a success listing numbered results, each with a title, a full `https://` URL and a snippet, and at least one URL belongs to the library's documentation site

#### Scenario: Redirect links resolved
- **WHEN** the search engine returns a result whose link points to its own redirect address
- **THEN** the result shows the destination URL, not the redirect address

### Requirement: Number of results
`web_search` SHALL accept an optional `max_results` between 1 and 10, default 5, and SHALL return at most that many results.

#### Scenario: Default count
- **WHEN** `web_search` is called without `max_results` and the search engine returns 10 results
- **THEN** 5 results are shown

#### Scenario: Out of range
- **WHEN** `web_search` is called with `max_results` 50
- **THEN** the tool does not run and the error result names `max_results`

### Requirement: No results
When the search engine returns no results, `web_search` SHALL return a success saying that nothing was found for the query.

#### Scenario: Nothing found
- **WHEN** a query returns no results
- **THEN** the result is a success saying no results were found for that query

### Requirement: Search errors
`web_search` SHALL return an error result, never crash, when the search cannot be done: the network fails or times out (after 20 seconds), the search engine answers with an HTTP error, or it refuses the request (for example, because of rate limiting). The message SHALL say what happened and, for refusals, suggest trying again later.

#### Scenario: Offline
- **WHEN** there is no network connection
- **THEN** the result is an error saying the search engine could not be reached

#### Scenario: Rate limited
- **WHEN** DuckDuckGo answers with a challenge page instead of results
- **THEN** the result is an error saying the search was refused and suggesting to try again later
