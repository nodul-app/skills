---
name: dynamic_params
description: Load before calling get_dynamic_node_parameters or when a node parameter is dynamic.
---

# Dynamic Parameters

## Algorithm

1. Read the node's `params` from `search_node_types`.
2. If it has a connection parameter, call `search_connections` with `connectionTypeAlias`.
3. Add the selected connection ID to `currentParameters`.
4. Call `get_dynamic_node_parameters` with `nodeTypeAlias` and the **complete** `currentParameters`.
5. Select required option keys, add them to the same object, and call again.
6. Continue until no newly returned field requires another dynamic selection.

Never send only the newest value; the server needs the full cascade context.

For selects, use the option key, not its human-readable label. Never invent an option.

## Google Sheets cascade

Node:

`__pd_google_workspace_drive_actions_sheets_v4_values_get_cells_all`

Step 1:

```json
{
  "nodeTypeAlias": "__pd_google_workspace_drive_actions_sheets_v4_values_get_cells_all",
  "currentParameters": {
    "access_token": "<connectionId>"
  }
}
```

Returned `drive_id`; `my_drive` was a valid option key.

Step 2:

```json
{
  "nodeTypeAlias": "__pd_google_workspace_drive_actions_sheets_v4_values_get_cells_all",
  "currentParameters": {
    "access_token": "<connectionId>",
    "drive_id": "my_drive"
  }
}
```

Returned `drive_item_id` with spreadsheet IDs as option keys.

Step 3:

```json
{
  "nodeTypeAlias": "__pd_google_workspace_drive_actions_sheets_v4_values_get_cells_all",
  "currentParameters": {
    "access_token": "<connectionId>",
    "drive_id": "my_drive",
    "drive_item_id": "<spreadsheetId>"
  }
}
```

Returned `worksheet_name`.

Step 4:

```json
{
  "nodeTypeAlias": "__pd_google_workspace_drive_actions_sheets_v4_values_get_cells_all",
  "currentParameters": {
    "access_token": "<connectionId>",
    "drive_id": "my_drive",
    "drive_item_id": "<spreadsheetId>",
    "worksheet_name": "Sheet1"
  }
}
```

Returned the optional static `range_a1_notation` field.

The same completed parameters successfully ran through `run_action_node_once` and read `Sheet1!A1:B5`.

## Validation behavior

- Missing required earlier fields returns a validation error instead of the next cascade fields.
- Some core nodes such as `http_request` do not have dynamic system metadata even if a select parameter is marked dynamic. Use this tool for node types that actually expose a dynamic cascade.
- A stale or unauthenticated connection may fail while loading options; try another valid saved connection or reauthenticate it.
