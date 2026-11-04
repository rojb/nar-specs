# Spec Delta

## Purpose

Defines the built-in `web_fetch` tool that lets the agent read a web page: which URLs it accepts, how content is converted for the model, its time and size limits, and the errors it reports.

## ADDED Requirements

### Requirement: Fetch a page
`web_fetch` SHALL accept a required `url` with the `http` or `https` scheme, download it following redirects, and return the page content preceded by its title (when it has one) and the final URL after redirects. It SHALL NOT require user approval.

#### Scenario: Documentation page
- **WHEN** `web_fetch` is called with the URL of a documentation page
- **THEN** the result is a success that starts with the page title and URL, followed by the page's text as Markdown

#### Scenario: Redirect followed
- **WHEN** the URL redirects to another address that returns a page
- **THEN** the content of the final page is returned and the shown URL is the final address

#### Scenario: Unsupported scheme
- **WHEN** `web_fetch` is called with `file:///etc/passwd` or `ftp://example.com`
- **THEN** the result is an error saying only http and https URLs are supported, and nothing is downloaded

### Requirement: HTML converted to Markdown
HTML pages SHALL be converted to Markdown keeping headings, paragraphs, lists, tables, code blocks and links. Links SHALL be made absolute. Scripts, styles, navigation menus, footers, forms and embedded frames SHALL be removed.

#### Scenario: Clean conversion
- **WHEN** a page has a `<nav>` menu, a `<script>`, an `<h1>Install</h1>`, a paragraph with a link to `/guide` and a `<pre>` code block
- **THEN** the result contains `# Install`, the paragraph with a link to the absolute URL of `/guide`, and the code block, and contains neither the menu text nor the script code

### Requirement: Other content types
Plain text, Markdown, JSON and XML responses SHALL be returned as text. Responses of other types (such as images, PDFs and archives) SHALL produce an error naming the content type.

#### Scenario: JSON response
- **WHEN** the URL returns `application/json`
- **THEN** the JSON text is returned

#### Scenario: PDF response
- **WHEN** the URL returns `application/pdf`
- **THEN** the result is an error saying the content type `application/pdf` is not supported

### Requirement: Limits
A fetch SHALL give up after 20 seconds and SHALL stop downloading after 5 MB. The returned text SHALL be subject to the standard tool output limit, so very long pages end with the truncation marker.

#### Scenario: Very long page
- **WHEN** the converted page is 80,000 characters long
- **THEN** the result contains the beginning of the page followed by the marker saying how many characters were omitted

#### Scenario: Slow server
- **WHEN** the server does not answer within 20 seconds
- **THEN** the result is an error saying the request timed out

### Requirement: Fetch errors
`web_fetch` SHALL return an error result, never crash, for network failures and HTTP error statuses. HTTP errors SHALL include the status code and the URL.

#### Scenario: Page not found
- **WHEN** the URL answers with HTTP 404
- **THEN** the result is an error containing `404` and the URL

#### Scenario: Unknown host
- **WHEN** the host name does not exist
- **THEN** the result is an error saying the URL could not be reached
