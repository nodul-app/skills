---
name: iterator_node
description: Load when working with iterator node.
---

### Iterator node

The Iterator processes a JSON array or object one item at a time. Use it when a node chain must run once per item. Prefer template operators for transformations that do not require per-item node execution.

#### Connector rule

The Iterator card has two different outputs:

- **Top cycle connector** — **body of the loop**. The node runs **once per item** (5 items → 5 runs).
- **Right connector** — **after the loop**. The node runs **once**, when the cycle has finished.

#### How to save through MCP

Both connectors can be saved through MCP.

| What we do | Result |
|---|---|
| Cycle body only (`parameters.nodes`, cycle root has empty `prevNodes`) | Works |
| Same save: cycle body **and** a right-side post-loop node | Works — cycle stays in `parameters.nodes`; post-loop has `prevNodes` on the Iterator |
| Later separate save: add a right-side post-loop node | Works — `parameters.nodes` is kept |

On `update_scenario` / `create_scenario`, cycle templates are only `{{$2.value}}` (digits after `$`). Never `{{$iterator_0.value}}` — that is not platform syntax. `get_scenario` may *show* a name; convert it to `$N` before writing.

- `data` is a string containing JSON or a runtime expression that resolves to an array/object.
- Arrays iterate over values; objects iterate over key-value pairs.
- Do not connect the cycle chain back to the Iterator.

#### MCP representation

Cycle nodes are **not nested objects** inside `Iterator.parameters.nodes`.

1. Keep every node, including cycle nodes, in the scenario's top-level `nodes` array.
2. Set `Iterator.parameters.nodes` to a string array containing the names of the cycle-root nodes:

```json
{
  "data": "[1,2,3,4,5]",
  "nodes": ["set_variables_0"]
}
```

3. A cycle-root node uses empty `prevNodes`.
4. Later nodes in the same cycle chain remain top-level and reference preceding cycle-node names through `prevNodes`.
5. A post-loop node is also top-level and references the Iterator through `prevNodes`. That is the **right** connector. It may be in the same `create_scenario` / `update_scenario` as the cycle body.
6. Multiple cycle roots are represented by multiple names in `parameters.nodes`.

`name` values are structural identifiers used by `prevNodes` and `parameters.nodes`. They are **not** runtime data references.

#### Required MCP creation sequence

1. `create_scenario`: trigger + Iterator + cycle nodes. Cycle roots listed in `parameters.nodes`. Cycle-root `prevNodes` empty. Optional: post-loop nodes with `prevNodes` pointing at the Iterator (right connector).
2. `get_scenario`: a real cycle keeps `parameters.nodes`. Cycle roots still have empty `prevNodes` (they must not point at the Iterator). Post-loop nodes **do** point at the Iterator.
3. `run_scenario_once`: `executedNodesCount` for the cycle node equals the item count; each post-loop node runs once.

`get_scenario` may **display** `{{$2.value}}` as `{{$iterator_0.value}}`. Do not write that back. Always send `{{$N.value}}`. Never send a cycle-root `prevNodes` pointing at the Iterator — that is the right connector, not the cycle.

#### Runtime data paths

Runtime expressions always reference the upstream node's **number**, never its `name`.

If the Iterator is node 2:

- Current index: `{{$2.index}}`
- Current value: `{{$2.value}}`
- Nested value: `{{$2.value.id}}`

Never write `{{$iterator_0.value}}`. `$` + node name is invalid. Use `{{$2.value}}` when the Iterator is canvas node 2.

If `data` comes from node 1:

```json
{
  "data": "{{$1.body.items}}"
}
```

#### Verified MCP example

This exact representation was verified through the live MCP. The Iterator executes once and `set_variables_0` executes five times.

```json
{
  "name": "Iterator runtime example",
  "nodes": [
    {
      "name": "run_once_0",
      "typeAlias": "run_once",
      "parameters": {
        "user_params": []
      }
    },
    {
      "name": "iterator_0",
      "typeAlias": "iterator",
      "parameters": {
        "data": "[1,2,3,4,5]",
        "nodes": ["set_variables_0"]
      },
      "prevNodes": {
        "run_once_0": {}
      }
    },
    {
      "name": "set_variables_0",
      "typeAlias": "set_variables",
      "parameters": {
        "variables": {
          "key": "{{$2.value}}"
        }
      },
      "prevNodes": {}
    }
  ]
}
```

Verified execution:

```json
{
  "executedNodesCount": {
    "run_once_0": 1,
    "iterator_0": 1,
    "set_variables_0": 5
  },
  "nodeOutputs": {
    "iterator_0": "{\"index\":4,\"value\":5}",
    "set_variables_0": "{\"key\":5}"
  }
}
```

#### Object iteration

Live output for `{"a":10,"b":20}` ended as `{"key":"b","value":20}` with `executedNodesCount` 2 on the cycle node. Inspect a real execution before assuming extra fields.

#### Data-flow limitations

- Cycle nodes can read upstream data and outputs from earlier nodes in their own cycle chain.
- Post-loop nodes cannot read outputs produced inside the cycle chain.
- A two-node cycle (`cycle_a` → `cycle_b` via `prevNodes`) works when the cycle root stays in `parameters.nodes`. Both nodes run once per item.

#### Testing

- Verify that `data` resolves to an array/object.
- Use `run_scenario_once` for end-to-end testing.
- Inspect `get_execution.executedNodesCount` to prove the cycle node ran once per item.
- Use non-destructive cycle actions until the loop behavior is verified.
