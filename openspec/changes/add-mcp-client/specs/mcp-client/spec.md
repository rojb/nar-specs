# Spec Delta

## Purpose

Defines how `nar` connects to external Model Context Protocol (MCP) servers over stdio and streamable HTTP, exposes their tools to the model as ordinary tools that go through human approval, keeps a failing server from affecting the rest of `nar`, and lets the user inspect servers with `/mcp`.

## ADDED Requirements

### Requirement: Connect configured servers at startup
When a request can run (interactive mode starts, or single mode has a prompt), `nar` SHALL connect to every server defined in `mcp_servers` whose `enabled` is not `false`, before the first request is handled. A server with `command` SHALL be started as a child process and spoken to over its standard input and output; a server with `url` SHALL be connected over streamable HTTP, sending the configured `headers` with every request. Servers SHALL be connected concurrently, each limited by its `startup_timeout`, so the total wait is not the sum of the servers. While connecting, `nar` SHALL tell the user on standard error which servers it is connecting to; when no server is enabled it SHALL print nothing and start no process. Disabled servers SHALL NOT be started or contacted. A child process SHALL inherit only a minimal set of environment variables (those needed to run programs) plus the server's configured `env`, so that unrelated secrets of the user's shell are not passed to it. A command that is a Windows script such as `npx` (`npx.cmd`) SHALL start the same way it does from a terminal.

#### Scenario: stdio server starts
- **WHEN** `[mcp_servers.demo]` has `command` pointing to a working MCP server and `nar` starts
- **THEN** the server is started as a child process, its tools are discovered, and the first request can use them

#### Scenario: HTTP server with a header
- **WHEN** `[mcp_servers.remote]` has a `url` and `headers = { Authorization = "Bearer abc" }`
- **THEN** every HTTP request `nar` sends to that server carries the header `Authorization: Bearer abc`

#### Scenario: Servers connect concurrently
- **WHEN** two enabled servers each need about 3 seconds to answer
- **THEN** both are ready after about 3 seconds, not about 6

#### Scenario: Disabled server
- **WHEN** a server has `enabled = false`
- **THEN** no process is started and no connection is made for it, and `/mcp` still lists it as disabled

#### Scenario: No servers
- **WHEN** `mcp_servers` is empty
- **THEN** `nar` starts without printing anything about MCP and without importing or starting anything for MCP

#### Scenario: Windows script command
- **WHEN** a server is configured with `command = "npx"` on Windows and `npx.cmd` is on the path
- **THEN** the server starts

#### Scenario: Shell secrets are not inherited
- **WHEN** the user's shell has `OPENROUTER_API_KEY` set and a stdio server is started without an `env` entry for it
- **THEN** the child process does not receive `OPENROUTER_API_KEY`

#### Scenario: Interrupt while connecting
- **WHEN** the user presses Ctrl+C while servers are connecting
- **THEN** `nar` stops connecting, leaves no child process running and exits with code 130

### Requirement: References to environment variables in server settings
In `command`, every item of `args`, every value of `env`, `url` and every value of `headers`, `nar` SHALL replace each `${NAME}` with the value of the environment variable `NAME` at the time the server is started. The configuration file itself SHALL NOT change. A reference to a variable that is not set or is empty SHALL make that server unavailable with a warning that names the server and the variable, and SHALL NOT start it, so a request is never sent with an empty credential. Values of environment variables SHALL NOT appear in warnings, in `/mcp` or in any other output.

#### Scenario: Token from the environment
- **WHEN** `LINEAR_API_KEY` is set to `lin_test` and a server has `headers = { Authorization = "Bearer ${LINEAR_API_KEY}" }`
- **THEN** the server receives the header `Authorization: Bearer lin_test`, and the file still contains `${LINEAR_API_KEY}`

#### Scenario: Missing variable
- **WHEN** a server uses `${LINEAR_API_KEY}` and the variable is not set
- **THEN** `nar` prints one warning that names the server and `LINEAR_API_KEY`, does not contact the server, and continues with the other servers and the built-in tools

#### Scenario: Reference in a stdio setting
- **WHEN** a stdio server has `env = { LINEAR_API_KEY = "${LINEAR_API_KEY}" }`
- **THEN** the child process receives the variable with the value from the user's environment

#### Scenario: Value is not printed
- **WHEN** a server fails to connect with a token taken from `${LINEAR_API_KEY}`
- **THEN** neither the warning nor `/mcp` contains the token

### Requirement: A server that cannot be used does not stop nar
When a server cannot be connected (the command does not exist, the process exits at once, the URL is unreachable, the credentials are rejected, `startup_timeout` is exceeded, or it does not speak MCP), `nar` SHALL print one warning line naming the server and the reason, SHALL mark the server as failed, and SHALL continue with the remaining servers, the built-in tools and the rest of the session. `nar` SHALL NOT exit because of a failed server. Output that a stdio server writes to its standard error SHALL NOT be mixed into the terminal; it SHALL be kept in a log file whose path `/mcp` shows for a failed server.

#### Scenario: Command does not exist
- **WHEN** a server has `command = "does-not-exist"`
- **THEN** `nar` prints a warning such as `warning: MCP server 'x' is not available: command 'does-not-exist' not found`, and a request that uses a built-in tool still works

#### Scenario: Server rejects the token
- **WHEN** a streamable HTTP server answers 401 to the connection
- **THEN** `nar` prints a warning naming the server and saying authentication was rejected, and does not print the token

#### Scenario: Server too slow
- **WHEN** a server does not finish connecting within its `startup_timeout`
- **THEN** `nar` warns that the server timed out, stops waiting for it, and the child process (if any) is not left running

#### Scenario: One failure among several
- **WHEN** two servers are configured and one of them fails
- **THEN** the other one is connected and its tools are registered

#### Scenario: Server noise stays out of the terminal
- **WHEN** a stdio server prints log lines to its standard error while running
- **THEN** those lines do not appear in the terminal or in the answer stream

### Requirement: Discovered tools are registered as mcp__server__tool
For each connected server `nar` SHALL list its tools and make each available to the model under the name `mcp__<server>__<tool>`, where `<server>` is the configured server name and `<tool>` is the server's name for the tool. Characters other than letters, digits and `_` SHALL be replaced by `_`. A name SHALL never exceed 64 characters and SHALL start with a letter, because that is what every supported model provider accepts. When the full name would be longer than 64 characters, the tool part SHALL be shortened and made unique with a short suffix derived from the original tool name. When two tools of one server would get the same name, both SHALL still be registered, the later one with a distinct suffix derived from its original name. The server SHALL always be called with the tool's original name. Names SHALL be stable: the same server and tool always produce the same registered name. Tool lists longer than one page SHALL be read completely. When `enabled_tools` is set, only the tools it names SHALL be registered; a name in `enabled_tools` that the server does not offer SHALL produce a warning.

#### Scenario: Plain registration
- **WHEN** server `linear` offers a tool `list_issues`
- **THEN** the model is offered a tool named `mcp__linear__list_issues`

#### Scenario: Characters not accepted by providers
- **WHEN** server `my-srv` offers a tool named `search.items v2`
- **THEN** the registered name is `mcp__my_srv__search_items_v2`, and calling it sends `search.items v2` to the server

#### Scenario: Name too long
- **WHEN** a server offers a tool whose name makes `mcp__<server>__<tool>` longer than 64 characters
- **THEN** the registered name is at most 64 characters, matches `^[a-zA-Z][a-zA-Z0-9_]*$`, and calling it reaches the right tool

#### Scenario: Two tools collide after sanitizing
- **WHEN** a server offers `get-item` and `get.item`
- **THEN** both are registered under different names, and each reaches its own tool

#### Scenario: Servers with the same tool name
- **WHEN** servers `a` and `b` both offer a tool `search`
- **THEN** `mcp__a__search` and `mcp__b__search` both exist and do not interfere with each other or with built-in tools

#### Scenario: Allow-list of tools
- **WHEN** a server offers 40 tools and `enabled_tools = ["list_issues", "get_issue"]`
- **THEN** only `mcp__<server>__list_issues` and `mcp__<server>__get_issue` are registered

#### Scenario: Allow-list names a missing tool
- **WHEN** `enabled_tools` contains `lst_issues` and the server has no such tool
- **THEN** `nar` warns that the server has no tool `lst_issues`, and the other listed tools are registered

#### Scenario: Paginated tool list
- **WHEN** a server returns its tools in three pages
- **THEN** the tools of all three pages are registered

### Requirement: Tool descriptions and schemas are passed through
The model SHALL receive the server's description of each tool and the server's input schema as the tool's parameters, so that it can call the tool correctly. The description SHALL say which server the tool belongs to, and a very long description SHALL be shortened to a fixed limit so that one server cannot fill the model's context. A tool without a description SHALL still get one that names the tool and the server. Arguments SHALL be checked against the tool's schema before the call, like every other tool; a call that does not match the schema SHALL return an error result to the model without contacting the server. A tool whose schema is not a valid JSON Schema object SHALL NOT be registered, and `nar` SHALL warn naming the server and the tool.

#### Scenario: Schema reaches the model
- **WHEN** a tool declares `{"type": "object", "properties": {"limit": {"type": "integer"}}, "required": ["limit"]}`
- **THEN** the tool definition given to the model has exactly that schema

#### Scenario: Invalid arguments are caught locally
- **WHEN** the model calls that tool with `{"limit": "ten"}`
- **THEN** the model receives an error result saying `limit` must be an integer, and the server is not called

#### Scenario: No description
- **WHEN** a server lists a tool without a description
- **THEN** the model's definition has a non-empty description that includes the server name and the tool name

#### Scenario: Very long description
- **WHEN** a tool's description is 20,000 characters long
- **THEN** the description given to the model is cut at the limit and ends with a marker saying it was shortened

#### Scenario: Broken schema
- **WHEN** a server lists a tool whose input schema is `{"type": "string"}`
- **THEN** that tool is not registered, `nar` warns naming the server and the tool, and the server's other tools are registered

### Requirement: Calling an MCP tool
When the model calls an MCP tool, `nar` SHALL send the call to the owning server with the original tool name and the model's arguments, wait for the result and give the model the text of the result. Text content blocks SHALL be joined in order. A block that is not text SHALL be replaced by a one-line description (kind, type, size or address) and its data SHALL NOT be given to the model. A result with no text and with structured content SHALL give the model the structured content as JSON. A result with nothing SHALL give the model the text `(no output)`. When the server marks the result as an error, or the call fails (protocol error, server gone, timeout), the model SHALL receive an error result with the reason, and `nar` SHALL keep running. A call SHALL be limited by the server's `tool_timeout`. The text given to the model SHALL be limited by the same output limit as every other tool. The call SHALL NOT block the terminal from being interrupted (see Cancelling a call).

#### Scenario: Text result
- **WHEN** the model calls `mcp__demo__echo` with `{"text": "hi"}` and the server returns the text `hi`
- **THEN** the model receives `hi` as a successful result and the terminal shows the call and a success line

#### Scenario: Several text blocks
- **WHEN** the server returns the blocks `one` and `two`
- **THEN** the model receives `one` and `two` in that order, separated by a blank line

#### Scenario: Image content
- **WHEN** the server returns a text block `chart` and an image block of 40,000 bytes
- **THEN** the model receives `chart` followed by a description such as `[image image/png, 40000 bytes, not shown]`, and none of the image data

#### Scenario: Resource link and embedded resource
- **WHEN** the server returns a resource link to `file:///a.txt`, an embedded text resource and an embedded binary resource
- **THEN** the model receives the link's address, the embedded text and a description of the binary resource without its data

#### Scenario: Structured content only
- **WHEN** the server returns no content blocks and structured content `{"count": 3}`
- **THEN** the model receives `{"count": 3}` as JSON text

#### Scenario: Tool reports an error
- **WHEN** the server marks the result as an error with the text `project not found`
- **THEN** the model receives an error result with the text `project not found`

#### Scenario: Protocol error
- **WHEN** the server answers the call with a protocol error such as invalid parameters
- **THEN** the model receives an error result that includes the server's message

#### Scenario: Call times out
- **WHEN** a tool takes longer than the server's `tool_timeout`
- **THEN** the model receives an error result saying the call timed out after that many seconds, and later calls to the same server still work

#### Scenario: Huge result
- **WHEN** a tool returns 200,000 characters of text
- **THEN** the model receives the beginning followed by the standard truncation note

#### Scenario: Server died during the session
- **WHEN** a stdio server's process exits and the model then calls one of its tools
- **THEN** the model receives an error result saying the server is no longer connected, `nar` keeps running, and `/mcp` shows the server as failed

### Requirement: MCP tools go through approval
Every MCP tool SHALL require approval, so that it is asked about at the `ask` and `auto-edit` levels and runs unasked only at `yolo`, with the "yes for the session" answer applying to that one tool. `nar` SHALL NOT treat a server's own hints (such as a read-only or destructive annotation) as permission to skip the question. The question's preview SHALL name the server, the tool's original name and the arguments as indented JSON, with non-ASCII text intact. Previewing a call SHALL NOT contact the server. A rejected call SHALL NOT reach the server.

#### Scenario: Ask at the default level
- **WHEN** the level is `ask` and the model calls `mcp__linear__list_issues`
- **THEN** the user sees the server `linear`, the tool `list_issues` and its arguments, and the server is called only after the user answers yes

#### Scenario: Rejected call
- **WHEN** the user answers no
- **THEN** the server is not called and the model receives the standard rejection result

#### Scenario: auto-edit does not cover MCP
- **WHEN** the level is `auto-edit` and the model calls an MCP tool
- **THEN** the user is asked

#### Scenario: yolo runs unasked
- **WHEN** the level is `yolo`
- **THEN** the call runs without a question

#### Scenario: Read-only hint is ignored
- **WHEN** a server marks a tool as read-only and the level is `ask`
- **THEN** the user is still asked

#### Scenario: Session approval is per tool
- **WHEN** the user answers `a` for `mcp__linear__list_issues`
- **THEN** later calls to that tool run without a question, and `mcp__linear__save_issue` is still asked about

### Requirement: Cancelling a call
Pressing Ctrl+C while an MCP call is running SHALL cancel the request the way it does for any tool (the terminal shows `cancelled` in interactive mode, single mode exits with code 130) within a second, SHALL NOT leave the terminal waiting for the server, and SHALL NOT disconnect the server: the next request can use its tools again.

#### Scenario: Cancel a slow call
- **WHEN** a tool takes 60 seconds and the user presses Ctrl+C after 2 seconds in interactive mode
- **THEN** `cancelled` is shown within a second, the prompt returns, and the next request can call the same server

### Requirement: Servers stop when nar stops
When `nar` ends (the user types `exit`, sends end-of-input, presses Ctrl+C at the prompt or during a request, a single-mode run finishes, or `nar` exits because of an error), it SHALL close every MCP connection and SHALL end every child process it started, without waiting more than a few seconds in total. A child process SHALL NOT keep running after `nar` has exited.

#### Scenario: Normal exit
- **WHEN** the user types `exit` with a stdio server connected
- **THEN** the server's process is gone shortly after `nar` returns to the shell

#### Scenario: Single mode
- **WHEN** `nar "<prompt>"` finishes a request that used an MCP server
- **THEN** the process exits with its usual code and the server's process is gone

#### Scenario: Server that ignores the close request
- **WHEN** a server process does not exit when its input is closed
- **THEN** `nar` ends it after a short grace period and still exits

### Requirement: The /mcp command
In interactive mode, `/mcp` SHALL print one block per configured server, in configuration order: its name, its transport (`stdio` or `http`), its status (`connected`, `failed` or `disabled`), and, for a connected server, the number of tools and their registered names. For a failed server it SHALL print the reason and, for a stdio server, the path of its log file. `/mcp <name>` SHALL print only that server. An unknown server name SHALL print an error that lists the configured names. When no server is configured, `/mcp` SHALL say so and name the configuration key and the files where servers are defined. `/mcp` SHALL NOT call the model, SHALL NOT contact any server and SHALL NOT print credentials, header values or environment values. The status SHALL reflect the current state, so a server that died during the session shows as failed.

#### Scenario: Servers and tools
- **WHEN** `linear` is connected with 3 tools and `jira` failed because its token was rejected, and the user types `/mcp`
- **THEN** the output shows `linear`, `http`, `connected`, 3 tools with their `mcp__linear__...` names, and `jira`, `http`, `failed` with the reason

#### Scenario: Single server
- **WHEN** the user types `/mcp linear`
- **THEN** only the `linear` block is printed

#### Scenario: Unknown server
- **WHEN** the user types `/mcp nope`
- **THEN** an error says there is no server `nope` and lists the configured names

#### Scenario: Nothing configured
- **WHEN** no server is configured and the user types `/mcp`
- **THEN** the output says no MCP servers are configured and explains where to define `[mcp_servers.<name>]`

#### Scenario: Disabled server listed
- **WHEN** a server has `enabled = false`
- **THEN** `/mcp` lists it with the status `disabled` and no tools

#### Scenario: Secrets are not shown
- **WHEN** a server uses `headers = { Authorization = "Bearer ${LINEAR_API_KEY}" }` and the user types `/mcp`
- **THEN** the output contains neither the token nor the header value

### Requirement: Demonstration with a real external service
The documentation SHALL describe how to connect Linear's official remote MCP server (`https://mcp.linear.app/mcp`) so that the user can ask `nar` to list the issues of a Linear project. It SHALL state what the user must provide (a Linear API key in an environment variable, or an interactive login through the documented fallback), which software is required for each way, and how to check the connection with `/mcp`.

#### Scenario: Documented configuration works as written
- **WHEN** the user copies the documented `[mcp_servers.linear]` block, sets `LINEAR_API_KEY` and starts `nar`
- **THEN** `/mcp` shows `linear` as connected with Linear's tools, and asking for the issues of a project makes `nar` call `mcp__linear__list_issues` after approval and answer from the result
