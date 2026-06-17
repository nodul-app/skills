---
name: nodul-scenario-builder
description: Expert guide for using nodul-mcp MCP tools effectively. Use when searching for nodes, managing scenarios, managing credentials or using any nodul-mcp tool. Provides tool selection guidance, parameter formats, and common patterns. IMPORTANT — Always consult this skill before calling any nodul-mcp tool — it prevents common mistakes like wrong nodeType formats, incorrect parameter structures, and inefficient tool usage. If the user mentions Nodul, scenarios, nodes, or connections and you have nodul-mcp MCP tools available, use this skill first.
---

# Nodul Scenario Builder

## Tool "create_scenario"

**Use when**: Creating a new scenario from scratch

**Syntax and example**:
```json
{
  "name": "Access a website using HTTPNode",
  "description": "A test scenario for demonstration purposes",
  "nodes": [
    {
      "name": "StartTrigger",
      "typeAlias": "run_once",
      "parameters": {}
    },
    {
      "name": "HTTPNode",
      "typeAlias": "http_request",
      "parameters": {
        "url": "https://example.com",
        "method": "GET"
      },
      "prevNodes": {
        "StartTrigger": {}
      }
    }
  ]
}
```

**Create scenario flow**:
1. Identify the specific applications and services you’ll need to implement the scenario.
2. Use the “search_node_types” tool to retrieve the aliases of nodes that fit the scenario’s flow. In the “query” parameter, provide a list of all the applications from step 1, using only one keyword for each.
3. If there are no suitable nodes for certain applications or services, use the “webhook” node for triggers and the “http_request” node for actions.
4. Start building the graph structure with a trigger.
5. When building the scenario, fill in the node parameters immediately. Specify optional parameters only if it is truly necessary.
6. For each node except the trigger, specify the “prevNodes” field with the alias of the one or several previous nodes.
7. Then call the “create_scenario” tool. If there are no validation errors, the ID of the created scenario will be returned.
8. Run the scenario using the “run_scenario_once” tool. For scenarios where the trigger is a webhook, run it asynchronously (“async”: true). After receiving a response with the execution ID, send a request to the URL of the created webhook.
9. If the “result” field in the response to the ‘run_scenario_once’ tool call is not “success,” follow the instructions in the troubleshooting section.

---

## TEMPLATE SYNTAX

Values may contain templates wrapped in "{{ ... }}".

### Accessing values from previous nodes

Let’s assume that the scenario has the following structure:
```json
{
  "name": "Echo scenario",
  "nodes": [
    {
      "name": "WebhookTrigger",
      "typeAlias": "webhook",
      "parameters": {
        "path": "test-path"
      }
    },
    {
      "name": "WebhookResponse",
      "typeAlias": "respond_to_webhook",
      "parameters": {
        "body": ""
      },
      "prevNodes": {
        "WebhookTrigger": {}
      }
    }
  ]
}
```
To access values from the output of previous nodes, use the following syntax:
- For a single value: {{$prevNodeName.\`body\`.\`output_key\`}}
- For an array element by index: {{$prevNodeName.\`output_key\`.\[0\]\.\`output_key\`}}

In the example above, set the value of the "body" field:
```json
{
  "body": "{{$WebhookTrigger.`body`.`output_key`}}"
}
```
During the execution of the scenario, the output value of the "body.output_key" will be set to the value "body" of the WebhookResponse node.

# Routing

## Understanding `prevNodes`

Scenario is a Directed Acyclic Graph (DAG). The edges of the graph are specified in the `prevNodes` field for each node.
For each edge, you can specify a condition that must be met for the node to be executed. If no condition is specified, 
the node will be executed in any case. A single node may include multiple edges (in the `prevNodes` fields of other nodes).
If none of the conditions are met, the node with the `fallback` flag set in `prevNodes` is triggered.

#### Example:
```json
{
  "name": "Echo scenario",
  "nodes": [
    {
      "name": "WebhookTrigger",
      "typeAlias": "webhook",
      "parameters": {
        "path": "test-path"
      }
    },
    {
      "name": "GetValuesInSheet",
      "typeAlias": "__pd_google_sheets_get_values",
      "parameters": {
        "access_token": "6a1024178cd196d0141573b5",
        "sheetId": "1YjtKgsUAxuTFhRmNxFHr9_U8qDWIa3v2KSj0lC5WPuw",
        "sheetName": "Sheet1"
      },
      "prevNodes": {
        "WebhookTrigger": {
          "condition": "{{$WebhookTrigger.query.param=\"some_value\"}}"
        }
      }
    },
    {
      "name": "GetAnotherValuesInSheet",
      "typeAlias": "__pd_google_sheets_get_values",
      "parameters": {
        "access_token": "6a1024178cd196d0141573b5",
        "sheetId": "1YjtKgsUAxuTFhRmNxFHr9_U8qDWIa3v2KSj0lC5WPuw",
        "sheetName": "Sheet1"
      },
      "prevNodes": {
        "WebhookTrigger": {
          "condition": "{{$WebhookTrigger.query.param=\"another_value\"}}"
        }
      }
    },
    {
      "name": "WebhookResponseFallback",
      "typeAlias": "respond_to_webhook",
      "parameters": {
        "body": "Unknown param"
      },
      "prevNodes": {
        "WebhookTrigger": {
          "fallback": true
        }
      }
    }
  ]
}
```

# Webhook

## What Webhook is

Webhook is a trigger node that serves as the entry point of a scenario. 
When a request is sent to the node's URL, the scenario starts executing.

## When to Use Webhook

- An external system needs to push data into Nodul in real-time (e.g., form submissions, payment events, GitHub pushes).
- You need the scenario to start instantly when something happens, not on a schedule.
- You're building inter-scenario communication where one scenario triggers another.

## How Webhook Works in Nodul

### Webhook Types

| Type                                         | Description                                                                                                                          |
|----------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| **App-specific webhooks (Instant triggers)** | Built into specific apps. Labeled "INSTANT" in module lists. E.g., `Slack/Triggers/New Mention`. Auto-configured with the app's API. |
| **Custom webhooks**                          | Generic HTTP endpoints via the Webhooks app. Accept any JSON/form data. You define the data structure.                               |

## How to Run and Test Scenario with Custom Webhook

1. Call the 'run_scenario_once' tool with the scenario ID and the 'async' parameter set to 'true'.
2. Then send a request to the URL of the created webhook. Use format `defaultWebhookURL/<webhook_path>`. The `defaultWebhookURL` can be found in the response from the SearchNodeTypes tool for alias = "webhook".
3. If there is no respond_to_webhook node in the scenario, the response will be "request accepted".

## Webhook node output
- **body** – the body of the webhook request. If the request is textual, it will be a string.
- **headers** – the headers of the webhook request.
- **query** – the query parameters of the webhook request.

## What should you know about Webhook?

- **URL is unique and secret.** The webhook URL acts as both address and authentication. Don't expose it publicly without additional validation.
- **One request = one execution.** Each HTTP request triggers one scenario run. Batch multiple items in a single request if you want to process them together (then use an Iterator to split).
- **Webhook queue.** If the scenario is busy, incoming webhook requests queue. They're processed in order when the scenario becomes available.

## Core Concepts Reference

When composing scenarios, consult these feature docs to understand how Nodul works. Read the relevant files before using these features in a module composition.

### Foundational
- **[Connections](connections.md)** — Authenticating modules with external services. OAuth, API keys, connection reuse.
- **[Dynamic Parameters](dynamic_params.md)** — Using dynamic parameters when configuring a node.
- **[JS Code](js_code_node.md)** — How to execute js code node in a scenario.
- **[Operators and functions](operators.md)** — How to use operators and functions in a scenario.
