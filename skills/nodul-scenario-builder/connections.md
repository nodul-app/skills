---
name: connections
description: Load when working with authenticating Nodul apps with external services via connections.
---

# Connections

## What Connections is

A connection is Nodul's way of authenticating with an external service. Before a scenario can interact with an app (e.g., Google Sheets, Slack, Stripe), it needs a connection that provides the credentials and permissions.

## When to Use Connections

- Most apps that interact with an external service require a connection (some public APIs like the Weather app do not).

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
