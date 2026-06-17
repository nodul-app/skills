---
name: Dynamic-params
description: Load before calling get_dynamic_node_parameters tool or if node type has parameter marked as "dynamic".
---

## Algorithm how to load dynamic parameters
1. Identify which connection types (aliases) are suitable for the node (usually for the `access_token` param).
2. Call the `search_connections` tool by providing those aliases.
3. Choose one connection from the result. Prefer the newest connection.
4. Call the `get_dynamic_node_parameters` tool by providing the connection id.
5. Tool returns new parameters for the node type. If the new parameters are marked as `dynamic`, 
call the `get_dynamic_node_parameters` tool again, including new parameters to the input (pass values with it).
6. Repeat step 5 until no more dynamic parameters are found.

## Example usage
1. Node type `__pd_google_sheets_get_values` has parameter `access_token` with type `connection` and field `"dynamic": true`.
    ```json
    {
      "nodeTypes": [
        {
          "alias": "__pd_google_sheets_get_values",
          "name": "Google sheets/Actions/Get Values",
          "description": "Get all values from a sheet.",
          "params": [
            {
              "key": "access_token",
              "title": "Connection",
              "type": "connection",
              "required": true,
              "options": [
                "google_sheets"
              ],
              "dynamic": true
            }
          ]
        }
      ]
    }
    ```
2. Calling the "search_connections" tool with the argument `"alias": ["google_sheets"]`. The response below:
    ```json
    {
      "connections": [
        {
          "id": "6a1024178cd196d0141573b5",
          "title": "sheets",
          "typeAlias": "google_sheets",
          "lastModifiedAt": "2026-06-06T20:01:48.311Z"
        }
      ]
    }
    ```
3. There is only one connection to choose – `6a1024178cd196d0141573b5`.
4. Call the `get_dynamic_node_parameters` tool by providing the connection id `6a1024178cd196d0141573b5`.
   #### Request:
   ```json
   {
     "nodeTypeAlias": "__pd_google_sheets_get_values",
     "currentParameters": {
       "access_token": "6a1024178cd196d0141573b5"
     }
   }
   ```
   #### Response:
   ```json
   {
     "parameters": [
       {
         "key": "drive",
         "title": "Drive",
         "type": "select",
         "required": false,
         "description": "Defaults to `My Drive`. To select a [Shared Drive](https://support.google.com/a/users/answer/9310351) instead, select it from this list.",
         "options": {
           "My Drive": "My Drive"
         },
         "dynamic": true
       },
       {
         "key": "sheetId",
         "title": "Spreadsheet",
         "type": "select",
         "required": true,
         "description": "The Spreadsheet ID",
         "options": {
           "1YjtKgsUAxuTFhRmNxFHr9_U8qDWIa3v2KSj0lC5WPuw": "Orders",
           "1s5FWnp6CtR6lfBkrfmgCAZE2QgnltRwWA8XCZa_CJng": "Test"
         },
         "dynamic": true
       }
     ]
   }
   ```
5. `drive` parameter is optional so ignore it. Then call the `get_dynamic_node_parameters` tool by providing the connection id `6a1024178cd196d0141573b5` and the new parameter `sheetId`.
   #### Request:
   ```json
   {
     "nodeTypeAlias": "__pd_google_sheets_get_values",
     "currentParameters": {
       "access_token": "6a1024178cd196d0141573b5",
       "sheetId": "1YjtKgsUAxuTFhRmNxFHr9_U8qDWIa3v2KSj0lC5WPuw"
     }
   }
   ```
   #### Response:
   ```json
   {
     "parameters": [
       {
         "key": "sheetName",
         "title": "Sheet Name",
         "type": "select",
         "required": true,
         "description": "Your sheet name",
         "options": {
           "Sheet1": "Sheet1"
         },
         "dynamic": true
       }
     ]
   }
   ```
6. Call the `get_dynamic_node_parameters` tool until only the non-dynamic parameters remain
   