---
name: mcp_trigger_node
description: Load when working with mcp_trigger node, or when the user asks to build a scenario that itself exposes an MCP server.
---

### MCP Trigger node
MCP Trigger node allows you to create MCP servers via scenarios (only tools). Each connected node to MCP trigger is a tool. An MCP tool can be made by several nodes connected with each other. The title and description of the first node in a chain of nodes is the title and description of the tool. The tool result is the result of the latest node in the chain, or the result of the first MCP Response node reached.
Important note! If you want to make an MCP server there cannot be any other trigger node in the scenario. Only one trigger, "MCP trigger", is allowed in a scenario when you want to make an MCP server.
You can find the MCP Trigger and MCP Response node aliases under the "AI Agent" app via `search_node_types`.

Note: this "MCP Trigger" node is a different concept from the MCP tools you are using right now to build this scenario. Building a scenario that contains an MCP Trigger node means you are building a **new, separate MCP server** — one that some other AI agent (possibly a completely different one, e.g. Claude Desktop or ChatGPT) will call later. It has nothing to do with the `create_scenario`/`update_scenario`/etc. tools you yourself are using.

#### Rules at a glance
- **Unique names.** Every first node of a tool chain must have a unique, descriptive name. A missing or duplicate name fails the run.
- **Tool Description.** Give each tool a **short but descriptive enough**, action-oriented description (e.g. "fetch current weather data"). Vague or overlong descriptions get the tool ignored or misused.
- **Param types are implicit.** You cannot declare a type inside `fromMCP()` — the schema type is derived from the node parameter that holds the template (see "Typing of tool parameters" below). No binary/file data through tools.
- **Agree tool schemas with the user first.** Do not silently invent tool inputs/outputs — clarify them with the user before building (see "Clarify tool input/output schemas with the user" below).

#### Clarify tool input/output schemas with the user
Same as with a webhook trigger, you must NOT silently decide by yourself which parameters each MCP tool accepts and what it returns.

**Inputs.** If the user's request does not make the tool inputs obvious, ask the user BEFORE building the tools what data the calling agent is actually going to pass into each tool at run time — at least a ROUGH input schema: which arguments the tool should accept, what each of them means. It does not have to be exhaustive or formally typed — an approximate list with short descriptions, or a realistic example call, is enough; you refine it into `fromMCP()` calls yourself. Skip the question only when the request already defines the inputs unambiguously (e.g. "a tool that takes a city name and returns the weather").

**Outputs.** For each tool, think about what the underlying node(s) actually return. If the raw output is (or may be) large or noisy — big JSON payloads, full API responses, HTML pages, long lists — ask the user to clarify what exactly should be returned to the calling agent (which fields / what summary), so you can trim the result with an MCP Response node instead of dumping the full payload into the caller's context. Do not pick the returned fields silently when the choice is not obvious from the request; propose what you would return and let the user confirm or adjust it. Small, already-compact outputs do not require this question.

Summarize the agreed inputs/outputs in chat before you build (tool name → arguments → what is returned) — optionally ask the user to confirm the plan, especially when the tools include write/destructive actions. There is no dedicated approval tool for this; a plain chat summary is enough.

#### MCP Trigger node fields
You must fill MCP Trigger required params:
- `description` — will be passed to the calling agent as "instructions" in the response to its initialize request.
- `version` — will be passed in the `serverInfo.version` response field of the initialize request.
You can set the title of the MCP Trigger node — this will be passed as the MCP server name in MCP responses.

#### API key authorization
Incoming MCP client connections to the activated scenario can be protected with an API key. The MCP Trigger node has two parameters for this:
- `use_authentification` (bool) — whether the check is enabled;
- `api_key` (string) — the key itself.

How it works:
- The values are read from the node and saved when the scenario is activated — changing them takes effect only after re-activating.
- When enabled, on every client connection the platform takes the `Authorization` header and compares it with the stored key as an **exact string match, one-to-one** — no `Bearer ` prefix is stripped, so the header value must be the bare key. A missing or non-matching header → the connection is rejected as unauthorized and no session is created.
- When disabled, the key is not checked at all — the prod URL is **public**. If the server exposes sensitive tools or write actions, warn the user and suggest enabling the API key.

When authorization is enabled, include the header in the client setup instructions (see "Transport & connecting clients"), e.g.:
- **Claude Code**: `claude mcp add --transport http <server-name> <url> --header "Authorization: <api_key>"`
- **Cursor** (`.cursor/mcp.json`):
```json
{ "mcpServers": { "<server-name>": { "url": "<url>", "headers": { "Authorization": "<api_key>" } } } }
```
- **Codex** (`~/.codex/config.toml`): add `http_headers = { "Authorization" = "<api_key>" }` under the `[mcp_servers.<server-name>]` section
- **OpenCode** / **GitHub Copilot**: add `"headers": { "Authorization": "<api_key>" }` to the server entry in the JSON config
- **OpenAI ChatGPT** and **Claude** (claude.ai / Desktop) connector UIs do not support custom headers — for these clients either keep the API key disabled or connect through a client that supports headers.

#### How tools work
When the scenario is saved and activated, the platform analyzes all connected first-level child nodes of the MCP Trigger and builds the set of tools. The tool's name and description come from the node's `name` and its `__$$internal__tool_description` parameter. To describe the parameters the tool accepts, use the `fromMCP()` function inside node parameters, e.g. `fromMCP("my_argument"; "My argument description")`. Here we declare that the tool accepts an argument `my_argument` with the description "My argument description". The system walks recursively through the whole subtree of that first-level child (including the node itself), finds every `fromMCP` call, and converts them into the tool's argument list. When the MCP tool is called by an external agent, the platform looks up the first-level node whose title matches the one in the request; the tool-call parameters become the result of the `fromMCP` calls in that tool's nodes. To return the result, use an MCP Response node — when execution reaches it, its result is sent as the tool's execution result. You cannot draw an edge from an MCP Response node to other nodes — it is final. A tool can also finish without an MCP Response node, in which case the result of one of the nodes at the end of that branch is used as the tool-call result — this variant is possible but not recommended.

MCP Response has only one parameter, `result`. Provide a template string value that will be resolved as JSON, e.g. `{{ $SomeNode.`body`.`data` }}` or `{ "result": {{ $SomeNode.`body`.`data` }} }`. Do not use a `jsonStringify` function for this — the data-access template resolves as JSON automatically.

Reaching MCP Response finishes the response path, but already-started parallel branches may still continue and produce side effects. Do not place unintended write actions on parallel branches under the assumption that MCP Response cancels them.

The MCP Trigger node itself decides which first-level node the scenario starts from. The system automatically sets a `condition` on each edge (each `prevNodes` entry) from the MCP Trigger, of the form `{{$McpTrigger.my_tool_name!=null}}` — it marks that data flows along this path only for that specific tool. You must NEVER set, change, or remove this auto-generated condition yourself: seeing it in the scenario is normal and expected — leave it untouched.

A tool is connected as a first-level child of the MCP Trigger. A tool can be a **single action node** or a **chain of action nodes** (the first node in the chain is a direct child of the MCP Trigger; the rest connect in sequence via `prevNodes` — invoking the tool runs the whole chain in order).
What the calling agent sees for each tool:
- **Name** — the `name` of the **first** node in the chain.
- **Description** — the `__$$internal__tool_description` parameter on that first node.
- **Arguments** — collected by scanning every `fromMCP()` expression across all node parameters in the tool's chain.
Data-flow rules: same as for a usual scenario — a tool node cannot access another node's data unless there is a `prevNodes` path from that node to the current one.

#### `fromMCP()` syntax
Use `fromMCP()` inside any input field of a tool node (URL, Request Body, Prompt, Text, headers, etc.) as a placeholder the calling agent fills at run time:
```
{{ fromMCP("paramName"; "description") }}
```
- Arguments are separated by a **semicolon** (`;`), not a comma.
- First argument: the parameter name on the tool node.
- Second argument: the parameter description — optional, but shown to the model and used in the function schema. Prefer including it.
- Only these two arguments exist — there is **no way to declare a type explicitly** in `fromMCP()`.

#### Typing of tool parameters
Tool parameters in the MCP `inputSchema` are typed, but **implicitly**. The platform derives the JSON Schema type not from the `fromMCP()` call itself, but from the type of the node parameter inside which the template is written:

| Node parameter type | Type in MCP schema |
| --- | --- |
| `int` | `number` |
| `bool` | `boolean` |
| `string`, `text`, `select`, `string_to_string`, `connection`, `string_to_any` | `string` |
| `string_array` | `array` |
| gateway (edge condition) expression | always `string` |
| unknown parameter type | empty `type` |

Important nuances:
- The mapping applies only when the `fromMCP()` template occupies the parameter value **as a whole** (custom/template mode). If `fromMCP()` is written inside a single element — e.g. one item of a `string_array` or one value of a `string_to_string` map — the argument is just a `string`.
- Complex schemas are not supported: no `enum`, no nested objects, no `items` for arrays. Each parameter carries only `type` and `description`. To confirm a node's exact parameter keys and types before writing a `fromMCP()` call, check the `params` array returned by `search_node_types` for that node's alias.

#### Example: MCP Server with HTTP tools
In this example we use the basic HTTP Request node to make requests to the internet.
```json
{
  "name": "Test MCP Server",
  "description": "A test scenario for demonstration purposes",
  "nodes": [
    {
      "name": "McpTrigger",
      "typeAlias": "mcp_trigger",
      "parameters": {
        "path": "fd4286e5-eb25-4b57-86ed-4c120c3df793",
        "description": "Here you can make post or get requests to the internet",
        "version": "0.0.1",
        "use_authentification": false,
        "api_key": ""
      }
    },
    {
      "name": "make_post_json_request",
      "typeAlias": "http_request",
      "parameters": {
        "__$$internal__tool_description": "Http POST request with content-type application/json",
        "url": "{{ fromMCP(\"url\"; \"The URL to send the request to\") }}",
        "method": "POST",
        "headers": { "Content-Type": "application/json" },
        "body": "{{ fromMCP(\"body\"; \"JSON body of the request\") }}"
      },
      "prevNodes": { "McpTrigger": {} }
    },
    {
      "name": "make_get_request",
      "typeAlias": "http_request",
      "parameters": {
        "__$$internal__tool_description": "Send an HTTP GET request to connect with external APIs and services",
        "url": "{{ fromMCP(\"url\"; \"The URL to send the request to\") }}",
        "method": "GET"
      },
      "prevNodes": { "McpTrigger": {} }
    },
    {
      "name": "RespondWithGetResult",
      "typeAlias": "mcp_response",
      "parameters": {
        "result": "{{ $make_get_request.`body` }}"
      },
      "prevNodes": { "make_get_request": {} }
    }
  ]
}
```
In this example an MCP server is created with 2 tools:
- `make_post_json_request` — send a POST request with a JSON body. It is not followed by an MCP Response node, so the full result of the HTTP Request node is sent as the response. This is not recommended, though, because the response may carry extra information that only pollutes the calling agent's context.
- `make_get_request` — send a GET request, followed by `RespondWithGetResult` (an MCP Response node), so the calling agent receives exactly the `body` of the response and nothing extra.

As stated above, you must not set or edit the `condition` the platform auto-generates on the root edges from `McpTrigger` (e.g. `{{$McpTrigger.make_post_json_request!=null}}` and `{{$McpTrigger.make_get_request!=null}}`) — seeing these in the scenario state is expected system behavior. On the edge between `make_get_request` and `RespondWithGetResult` a `condition` can be set if you need one.

#### Testing an MCP Trigger scenario
Important! When `fromMCP()` is used inside a node, that node cannot be tested in isolation with `fromMCP()` still in place — it will error, because the value is only supplied when an actual MCP tool call reaches the node from outside. There is no tool in this toolset that invokes a single MCP tool inside a scenario end-to-end (unlike a regular in-app builder session, this MCP-based workflow does not have a way to simulate an external MCP client call).

To build and verify a tool chain:
1. Add the nodes to the scenario.
2. Fill them with literal test values WITHOUT `fromMCP`, as if the calling agent had already supplied the arguments.
3. Run the chain's action nodes individually via `run_node_once` and inspect their outputs. Do not add a second trigger or claim an end-to-end MCP test through `run_scenario_once`.
4. Once the chain behaves correctly, replace the relevant parameter values with `fromMCP(...)` where they need to be supplied by the calling agent.
5. Activate the scenario (see "Deployment" below). Full end-to-end verification — actually calling the tool as an external MCP client would — has to happen either by asking the user to connect their own MCP client to the activated URL and try it, or by the user manually testing it in the platform's own builder UI (which does have a request-simulation tool for this). Say so explicitly to the user instead of claiming the tool chain is fully verified.

#### Transport & connecting clients
The MCP server is exposed over the **Streamable HTTP** transport. When you give the user the URL where their MCP server is available, always say it is an HTTP-transport MCP server, and ideally include ready-to-use setup instructions for popular MCP clients, for example:
- **OpenAI ChatGPT**: Settings → Connectors → Advanced → enable Developer mode, then Connectors → Create → paste `<url>`
- **Claude** (claude.ai / Claude Desktop): Settings → Connectors → Add custom connector → paste `<url>`
- **Claude Code**: `claude mcp add --transport http <server-name> <url>`
- **Codex**: `codex mcp add <server-name> --url <url>`, or add to `~/.codex/config.toml`:
```toml
[mcp_servers.<server-name>]
url = "<url>"
```
- **OpenCode**: add to `opencode.json` (project-level or `~/.config/opencode/opencode.json`):
```json
{ "mcp": { "<server-name>": { "type": "remote", "url": "<url>", "enabled": true } } }
```
- **GitHub Copilot** (VS Code): add to `.vscode/mcp.json`:
```json
{ "servers": { "<server-name>": { "type": "http", "url": "<url>" } } }
```
- **Cursor**: add to `~/.cursor/mcp.json` (or project-level `.cursor/mcp.json`):
```json
{ "mcpServers": { "<server-name>": { "url": "<url>" } } }
```
Provide the same kind of snippet for any other client the user mentions (VS Code, Windsurf, etc.).

#### Deployment
When the work on the MCP server is finished, the scenario must be activated via `activate_scenario` — only after activation does the MCP server become available at the **production URL**. Until then the user's MCP clients cannot reach it.
- If the scenario was **empty** when you started (you built the MCP server from scratch): call `activate_scenario` right away — nothing was live before, and nothing runs until a client calls the URL. You may still mention in chat what you are about to activate, but a blocking approval step is not required for a from-scratch build.
- If you **edited an existing scenario**: mention in chat that you are about to activate your changes and give the user a chance to object first, since activation replaces the currently live version and may affect functionality already in use by real MCP clients. This is a text-level courtesy, not a hard tool gate — there is no dedicated approval tool in this toolset.

After activating, give the user the prod URL together with the client setup instructions above.
