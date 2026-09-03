---
name: connections
description: Load when working with authenticating Nodul apps with external services via connections.
---

# Connections

## What Connections is

A connection is Nodul's way of authenticating with an external service. Before a scenario can interact with an app (e.g., Google Sheets, Slack, Stripe), it needs a connection that provides the credentials and permissions.

## When to Use Connections

- Most apps that interact with an external service require a connection (some public APIs like the Weather app do not).
- First inspect the node's `params` from `search_node_types`. If there is no connection-type parameter, do not call connection tools.
- Nodes described as **Plug and Play** / **PnP** do not require the user's own connection; skip `search_connections` and `create_connection` for them.

## AUTHORIZATION HANDLING (HIGHEST PRIORITY — ALWAYS FIRST)

Authorization/connection parameters MUST be handled **FIRST**, before any other parameters.

**Execution order:**
1. **FIRST** — Check if there are any authorization/connection parameters in the node.
2. **SECOND** — If authorization parameters exist and have available options, run 'search_connections' providing aliases of available connections.
3. **THIRD** — Only after authorization is set, proceed to configure other parameters.

**Selection rules for authorization data:**
- **If a credential is already set and valid — do NOT change it.** Skip authorization configuration entirely and proceed to other parameters.
- If only one credential is available and no credential is currently set — use it immediately without asking.
- If multiple credentials are available — automatically select the most appropriate one:
  - Prefer the one with a name that best matches the current integration/service context.
  - If no clear match, prefer the most recently added or the first in the list.
- If a node accepts several authorization types, prefer the normal OAuth connection for that app. Use personal-app/client credential variants only when the user explicitly asks for them or OAuth is unsuitable.
- If a credential is already set and valid — do not change it unless there is an explicit reason.
- After auto-selecting a credential, briefly inform the user which connection was used (e.g., "I've selected the 'My Gmail Account' connection for this node.").

**Ask about authorization ONLY when:**
- The authorization parameter is required AND no credentials are available in the options list at all.
- In this case, inform the user that authorization is needed and explain how to add it.

Do NOT ask the user to choose between available credentials — make the choice yourself.

## Connection Lifecycle

1. **Create a connection** — authenticate with the external service (OAuth flow, API key, or other method depending on the app).
2. **Assign to modules** — each module that uses the app references the connection.
3. **Reuse across modules** — multiple modules for the same app can share one connection.
4. **Reuse across scenarios** — connections are organization-level resources, shared across all scenarios in a team.

## Authentication Methods

| Method           | Description                                                      | Common apps                        |
|------------------|------------------------------------------------------------------|------------------------------------|
| **OAuth 2.0**    | Redirects to the service for authorization. Tokens auto-refresh. | Google, Slack, HubSpot, Salesforce |
| **API Key**      | Simple key-based auth. Entered directly.                         | OpenAI, Anthropic, many REST APIs  |
| **Basic Auth**   | Username/password pair.                                          | Legacy APIs, some databases        |
| **Custom/Token** | Service-specific token or configuration.                         | Webhooks, custom HTTP modules      |

## Connection Per Scenario

When composing a scenario, note which apps are involved. Each distinct app needs at least one connection. Multiple modules from the same app typically share a connection.

## Searching connections: `search_connections`

Use the live parameter name:

```json
{
  "connectionTypeAlias": ["google_sheets"]
}
```

The older `alias` key is not accepted by the current live schema.

## Creating a new connection: `create_connection`

Use this tool only when `search_connections` found no usable existing connection for the required alias(es) (per the Authorization Handling rules above).

**Input**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `typeAlias` | string | yes | Connection type alias (same alias space used by `search_connections`) |
| `title` | string | yes | Display name for the connection |
| `parameters` | object | no | Connection parameters (e.g. API key/secret for non-OAuth connections) |

**Output**

| Field | Description |
|---|---|
| `connectionId` | ID of the created connection — use it in the node's connection-type parameter (often `access_token`; confirm the exact key via `search_node_types`) |
| `url` | Authorization URL, returned for OAuth-based services when the client does not support elicitation |

**Behavior notes**:
- For **OAuth-based services** (Google, Slack, HubSpot, etc.), `create_connection` does not fully finish the auth itself — it returns a `url`. **Immediately post that URL in chat** as a clickable link. Do not rely on elicitation or assume the user will find it elsewhere. Wait until they confirm authorization is done before using the connection.
- For **API-key/token-based services**, pass the credential(s) in `parameters` and the connection is created immediately (no `url` returned).
- Never invent or ask the user to paste a raw credential into chat so you can embed it elsewhere — pass it directly as a `create_connection` parameter, and never echo it back afterward.
- After creating the connection, reference it by `connectionId` in the exact connection-type parameter returned by `search_node_types` (same way an existing connection from `search_connections` would be referenced).

## Reauthenticating: `reauthenticate_connection`

```json
{
  "connectionId": "existing-connection-id"
}
```

The tool returns a `url` when the client does not support elicitation. **Immediately post that URL in chat** as a clickable link and wait until the user confirms authorization is done. Do not only "open it" silently.
