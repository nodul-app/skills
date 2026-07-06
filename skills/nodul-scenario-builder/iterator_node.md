---
name: iterator_node
description: Load when working with iterator node.
---

### Iterator node
The Iterator is a special action node that processes a collection **one element at a time, in order**. You give it an array or object, and it feeds each item into a connected node chain, running that chain once per item. Use it to loop over lists, query results, or the key–value pairs of an object.
#### When to use the Iterator node
Use the Iterator whenever you need to **run a chain of nodes per item** in a collection (e.g., send an HTTP request for each record, write each row to a sheet, message each user). It is the deterministic, node-native way to loop.
Prefer template operators/functions over an Iterator when you only need to **transform** an array in place and don't need to run nodes per item — e.g. `map`, `sort`, `distinct`, `merge`, `slice`, `join`, `sum`. Reach for a JavaScript node only when the per-item logic genuinely exceeds both operators and what a node chain can do.
#### Rules at a glance
- **Single parameter.** The Iterator has one parameter, `data` (string). Pass a JSON array, a JSON object, or an expression resolving to one.
- **Array vs object.** A JSON **array** iterates over its elements; a JSON **object** iterates over its **key–value pairs**.
- **Two connectors, two meanings.** The `cycle` (top) handle runs **once per item**; the `output` (right) handle runs **once, after all iterations finish**.
- **Read the current item from the Iterator itself.** Inside the cycle chain, reference the current value/index via `{{ $<iteratorNodeName>.`...` }}`.
- **Isolation.** Nodes placed AFTER the Iterator (on the `output` handle) cannot read data produced inside cycle-chain nodes.
#### The `data` parameter
The Iterator has one parameter, `data`, of type string.
You can put a plain JSON string containing the array of items:
```
{
  "data": "[{\"id\": 1, \"name\": \"Item 1\"}, {\"id\": 2, \"name\": \"Item 2\"}, {\"id\": 3, \"name\": \"Item 3\"}]"
}
```
Or a custom expression that pulls the array from a previous node:
```
{
  "data": "{{ $webhook.`body`.`data` }}"
}
```
You can also pass a JSON **object** instead of an array. In that case the Iterator loops over the object's **key–value pairs** rather than array elements.
#### Connectors (cycle vs output)
The Iterator exposes two outgoing handles, and the difference between them is the most important thing to get right:
- **`cycle` handle (top connector)** — the loop body. Nodes attached here run **for each element**, as many times as there are items in `data`. Attach the node chain that should process a single item (HTTP Request, Set Variables, JavaScript, etc.).
- **`output` handle (right connector)** — runs **only once, after all iterations are done**. Use it for "when everything is processed" steps, e.g. sending a Webhook Response or a final summary. It is optional.
  You can attach any number of node chains to the `cycle` handle — each chain runs once per item. You may attach up to two chains in parallel directly to the `cycle` handle; for each item the Iterator fans out to both branches simultaneously.
#### Reading the current item
Every node in the cycle chain can read the current iteration's data directly from the Iterator node. Assuming the Iterator's node name is `<iteratorNodeName>`:
- Current index: `{{ $<iteratorNodeName>.`index` }}`
- Current value: `{{ $<iteratorNodeName>.`value` }}` (drill into fields, e.g. `{{ $<iteratorNodeName>.`value`.`id` }}`)
  This iterator output is produced at runtime, after the Iterator runs — so it is only resolvable inside nodes on the cycle chain (or downstream, once the platform has executed the Iterator at least once).
  For object iteration (key–value pairs), the per-iteration fields differ from the array case.
#### Data-flow rules
- Nodes in a cycle chain can read data from previous nodes **within the same chain** and from nodes **upstream of the Iterator**.
- Every cycle-chain node can additionally read the current item from the Iterator itself (see above).
- Nodes placed AFTER the Iterator (on the `output` handle) **cannot** read data from cycle-chain nodes — those run inside the loop and are not exposed to the post-loop branch.
#### Example: webhook → iterate items → HTTP request → respond
A workflow that iterates over a list of items and sends a request for each one, then responds once after the loop.
NOTE: as in the **[ai_agent_node](ai_agent_node.md)** skill, the `paramValues` shown inline below are for illustration only.
```
{
  "nodes": [
    {
      "name": "Webhook",
      "typeAlias": "webhook",
      "parameters": {
        "path": "c7a98c62-0aaf-415e-85bf-736b1ecb4a96"
      },
      "prevNodes": {}
    },
    {
      "name": "Iterator",
      "typeAlias": "iterator",
      "parameters": {
        "data": "{{ $Webhook.`body`.`items` }}"
      },
      "prevNodes": {
        "Webhook": {}
      }
    },
    {
      "name": "HttpRequest",
      "typeAlias": "http_request",
      "parameters": {
        "url": "https://example.com/log-item",
        "method": "POST",
        "headers": {
          "Content-Type": "application/json"
        },
        "body": "{\"itemId\": \"{{ $Iterator.`value`.`id` }}\", \"itemName\": \"{{ $Iterator.`value`.`name` }}\" }"
      },
      "prevNodes": {}
    },
    {
      "name": "RespondToWebhook",
      "typeAlias": "respond_to_webhook",
      "parameters": {
        "status": "200",
        "headers": {
          "Content-Type": "application/json"
        },
        "body": "{ \"success\": true }"
      },
      "prevNodes": {
        "Iterator": {}
      }
    }
  ]
}
```
Here the workflow starts from the `webhook` trigger and runs the `iterator` node. The `http_request` node is attached to the Iterator's `cycle` handle, so it runs once per item, reading the current item via `{{ $Webhook.`value`.`id` }}` and `{{ $Webhook.`value`.`name` }}`. The `respond_to_webhook` node is attached to the Iterator's `output` handle, so it runs only once after all iterations finish, returning the final response to the caller.
