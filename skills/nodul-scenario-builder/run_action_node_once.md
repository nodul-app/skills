---
name: run_action_node_once
description: Load when the user wants a one-off action result in chat without building a scenario — run_action_node_once and its supporting tools.
---

# Run a single action (`run_action_node_once`)

Use this skill when the user wants **one catalog action executed immediately** and the result back in chat — **without** creating a scenario on the canvas or in the Scenarios list.

The public [MCP Tools Reference](https://docs.nodul.ru/mcp-server/tools) may temporarily lag behind this newly released tool. The schema and behavior below follow the current reference supplied with this skill package.

## When to use this mode vs a scenario

| User intent | Use |
|---|---|
| One-off: send a Slack message, add a sheet row, create a calendar event, look something up | **`run_action_node_once`** (this skill) |
| Recurring / triggered automation, multi-step graph, webhook/MCP/schedule | **`create_scenario`** + scenario tools (see `SKILL.md`) |
| Debug one step of an **existing** scenario | **`run_node_once`** (needs `scenarioId` + `nodeName`) — not this tool |

Do **not** create a scenario only to run a single action once. Prefer `run_action_node_once`.

## Tools used in this mode

| Tool | Role |
|---|---|
| `search_node_types` | Find the action (`typeAlias`) and its `params` schema |
| `get_dynamic_node_parameters` | Load fields that appear after initial values (e.g. sheet list after spreadsheet) |
| `search_connections` / `create_connection` | Reuse or create an authorization |
| `run_action_node_once` | Run the action and return the result in the same call |

## `run_action_node_once` — schema

**Input**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `nodeTypeAlias` | string | yes | Node type identifier from `search_node_types` |
| `parameters` | object | no | Node configuration; required node fields are validated |
| `session` | string | no | Session for references to earlier one-off runs. Default: `default` |
| `async` | boolean | no | Return only the execution ID. Default: `false` |
| `compactOutput` | boolean | no | Keep only the first five values of every output array. Default: `true` |

**Output**

- The action result in the **same** response (no separate scenario to open).
- If the node needs an app connection and none work, the response includes a **link to create a new authorization**.

**Notes**

- Distinct from `run_node_once`: **no `scenarioId`**. You do not select a node on the canvas.
- Existing connections are tried first, **most recently updated first**. If none work, complete the authorization link, then call the tool again.
- Required parameters are validated **before** the action runs.
- The run is cancelled after **100 seconds**.
- The helper scenario is **hidden** from the Scenarios list. Do not promise that the run appears in the user's normal scenario History or Statistics.

## Recommended flow

1. **Find the action** — call `search_node_types` with a focused string `query`, select an app, then call it with `app`. Read `alias`, `params`, and descriptions. Never invent a `nodeTypeAlias`. Core nodes (`set_variables`, `js_code`, …) go to step 4 after filling params. Service actions (Gmail, LinkedIn, Sheets, …) go through auth and cascade first.
2. **Authorization first** (if the node has a connection / `access_token` / similar field):
   - Call `search_connections` with `connectionTypeAlias`.
   - Prefer an existing connection (newest first if trying several). If one fails after `access_token` is set, try the others in that same newest-first order.
   - If none work or none exist — call `create_connection`, surface any returned `url` to the user for OAuth, wait until they confirm, then continue.
3. **Cascade parameters** — set what you know, then call `get_dynamic_node_parameters` with `nodeTypeAlias` + `currentParameters`. Repeat until no further dependent fields remain. See `dynamic_params.md`.
4. **Run** — call `run_action_node_once` with `nodeTypeAlias` and the filled `parameters`. For large row-like output keep `compactOutput: true` (arrays truncated to five items).
5. **On auth failure** — give the user the authorization link from the response; after they finish, call `run_action_node_once` again with the same parameters (or a newly created connection id).

## Authorizations

Same authorizations as in the visual builder.

- Try **existing** authorizations for that app, newest first.
- If none work, send a create-authorization link; finish it in the browser, then ask the agent again.
- Until the authorization is ready, the action will keep failing — do not invent credentials or paste secrets into node fields as plaintext chat echoes.

## Example user prompts (one prompt ≈ one action)

```
Send “Deploy is live” to the #ops channel in Slack.
```

```
Create a Google Calendar event tomorrow at 15:00 titled “Sync with design”.
```

```
Add a row to the Leads sheet: name Anna, email anna@example.com.
```

If the user asks for several independent things at once, run **several** `run_action_node_once` calls — one per action.

## Limits

- Cancelled after **100 seconds**.
- Missing **required** fields stop the call before the action runs.
- Nothing appears in the Scenarios list / canvas for a successful one-off run.
- Normal History/Statistics visibility is currently unconfirmed; an execution ID alone does not prove UI visibility.

## Core vs app actions

1. Determine whether this is a **core** node (`set_variables`, `js_code`, `http_request`, …) or an **app** action (Gmail, LinkedIn, Slack, …).
2. Core **action** nodes: fill required parameters from the schema, then call `run_action_node_once`. Do not use this tool for trigger-only node types such as webhook or schedule.
3. App actions: handle connections + dynamic parameters first (steps above), then run.
