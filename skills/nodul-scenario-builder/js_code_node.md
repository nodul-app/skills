---
name: js_code_node
description: Load when working with js_code node.
---

### JavaScript code node
The JavaScript code node runs custom JavaScript to handle logic that native nodes and template expressions cannot express.
#### When to use the JS code node
Use the JS code node only when the task genuinely needs custom code. Before writing JS, check the simpler options first:
- Can the **Template Evaluator** do it? Use a custom expression for value transforms, conditions, and string/array/date/number helpers.
- Is it a per-item loop over a list? Use the **Iterator** node.
- Is there a **native action node** for the service? Prefer it.
  Reach for JS only for custom logic (complex transforms, multi-step API calls, custom parsing) that the above cannot cover.
#### Runtime environment
**Default runtime**: Bun.
##### Switching runtime (`use_bun`)
Control the runtime with the `use_bun` parameter:
- `use_bun: true` (default) — code runs on the **Bun runtime**.
- `use_bun: false` — code runs on the **Node.js runtime**.
  Note that the `use_bun` parameter is **not** present in the parameters schema. You must set it manually.
  When to switch to Node (`use_bun: false`):
- A required npm package or Node core module is not supported on Bun, or behaves differently. Bun is largely Node-compatible, but some libraries may behave differently — always check compatibility.
  Bun specifics:
- Bun ships extra built-ins such as `bun:sqlite`.
- Server creation (`Bun.serve`) and listening on ports are not supported.
  Neither runtime supports TypeScript/JSX syntax — write plain JavaScript only (no type annotations, interfaces, or generics).
#### Runtime facts & limits
- **`run()` argument object.** The injected argument object exposes `{ execution_id, data, store, db }`. You normally only need `data`; destructure the others when required. `data` is **not** automatically populated with every previous node output. Declared `@CustomParams` are available by key (strings, numbers, connections — **never a file**). An upstream **file** is only available via a hardcoded accessor in the source, for example `data["{{$3.result.fileInfo.content}}"]`. There is no CustomParams type that can take file content.
- **Variables via `store`** (all calls are `async`; values become available to other nodes after the JS node runs once):
- Scenario-local (string values only, scoped to the current scenario):
```javascript
await store.setVariable("MyVar", "value");
const v = await store.getVariable("MyVar");
```
- Global (String, Object, or Number; shared/persistent across the workspace):
```javascript
await store.setGlobalVariable("MyGlobal", { key: "value" });
const g = await store.getGlobalVariable("MyGlobal");
await store.deleteGlobalVariable("MyGlobal");   // permanent
const list = await store.listGlobalVariables(); // [{ key, type, editable, created_at, last_modified_at }]
```
Local variables accept strings only; global variables also accept objects and numbers.
- **Limits.** Maximum execution time is **2 minutes**. The environment is isolated: no listening ports, HTTP/WebSocket servers, or background daemons. Split heavy work across multiple JS nodes.
#### Code structure (MANDATORY)
Every JavaScript node must export a default `async` function named `run` that destructures `data` and returns an object:
```javascript
export default async function run({ data }) {
return {
// your result here
    };
}
```
This shape is required: the function must be `export default async`, named `run`, destructure `data` (the field you normally need), and return an object. `execution_id`, `store`, and `db` may be additionally destructured when needed. `run({ data })` is the canonical form.
Use `axios` for all HTTP requests.
**Critical requirements**:
- ALWAYS return an object (even if empty: `{}`).
- ALWAYS use `export default`.
- ALWAYS make the function `async`.
- NEVER return primitive values directly.
- NEVER use non-object return types.
#### NPM packages
- Any npm package can be imported — packages are auto-installed on save.
- Never ask the user to install packages manually. Simply import what you need.
- Pin a version with `@` when you need one, e.g. `import _ from 'lodash@4.16.6'` or `import axios from 'axios@^1.2.0'`.
- Use `import`, never `require`. Use `axios`, never `fetch`.
- If you run a node before installation finishes you get `Dependency installation is not yet completed` — wait a few seconds and retry.
#### Returning Files (MANDATORY FORMAT)
**This is the ONLY valid way to return files. Any other format will fail.**
**Available functions** (both are GLOBAL, never import them):
- `file(filePath)` — returns a single file from the specified path (string).
- `files(filePaths)` — returns multiple files from the specified paths (array of strings).
  **3-step process for single file**:
1. **Write to temporary filesystem**:
```javascript
import fs from "fs";
fs.writeFileSync("filename.ext", content);
```
2. **Return using global `file()` wrapper**:
```javascript
return {
file: file("filename.ext")
};
```
**Complete example (single file)**:
```javascript
import fs from "fs";
export default async function run({ data }) {
const content = "Hello, World!";
fs.writeFileSync("output.txt", content);
return {
file: file("output.txt")
    };
}
```
**Critical notes**:
- `file()` and `files()` are GLOBAL functions — never import them.
- ALWAYS write to filesystem first with `fs.writeFileSync()`.
- ALWAYS wrap filename(s) with `file()` or `files()`.
#### Returning Multiple Files
Use the `files()` function with an array of paths:
```javascript
import fs from 'fs';
export default async function run({ data }) {
fs.writeFileSync('file1.txt', 'content 1');
fs.writeFileSync('file2.txt', 'content 2');
fs.writeFileSync('file3.txt', 'content 3');
return {
file: file('file1.txt'),                  // single file
files: files(['file2.txt', 'file3.txt'])  // multiple files
    };
}
```
#### CRITICAL: First-Level Nesting Only
The `file()` and `files()` functions **ONLY work at the first level** of the returned object.
```javascript
// Correct - first level
return {
file: file('output.txt'),
files: files(['a.txt', 'b.txt']),
otherData: "some value"
};
// WRONG - nested deeper than first level (will NOT work!)
return {
object: {
file: file('output.txt'),          // TOO DEEP
files: files(['a.txt', 'b.txt'])   // TOO DEEP
    }
};
```
#### Reading Files from Previous Nodes (STRICT)

**There is no CustomParams type for file content.** Putting `{{$3.result.fileInfo.content}}` (or any `...content` path) into a custom parameter **breaks** the node. Custom params may be `string`, `int`, `connection`, etc. They cannot receive a file and pass it into `data`.

The file accessor **must be hardcoded in the source**, in this format (middle segments follow the real upstream output; the leaf is always `content`):

```javascript
const webpContent = data["{{$3.result.fileInfo.content}}"];
```

Equivalent numbered access with backticks:

```javascript
const webpContent = data["{{3.`result`.`fileInfo`.`content`}}"];
```

Then decode bytes with **`latin1`** (any other encoding corrupts the file):

```javascript
if (!webpContent) {
  throw new Error("File content not found.");
}
const imageBuffer = Buffer.from(webpContent, "latin1");
```

When a **JavaScript** node returns `file: file("name.csv")`, the next JS node reads it as **`data["{{$2.file.content}}"]`** (node 2 = the JS that returned the file). There is no `result` wrapper. The official CSV docs example `{{2.result.file.content}}` is for nodes whose output is nested under `result` — a JS `file()` return is top-level `file.content`.

`@CustomParams` stays valid for email body, recipient, Gmail connection, API keys — not for the file.

**Forbidden**

```javascript
/** @CustomParams
{ "image_file": { "type": "string", "title": "Image file" } }
*/
// then data.image_file = "{{$3.result.fileInfo.content}}"  — this will break
```

**Mandatory sequence when you also need to return a new file to downstream nodes:** read via `data["{{$N....content}}"]` → bytes (`Buffer.from(..., "latin1")` **or** `fs.readFileSync` if the accessor is a temp path) → modify → `fs.writeFileSync` → return `file(...)` / `files(...)` at the first level only.

Official Handling Files example (temp path + CSV transform + `file()`). Same rule: accessor hardcoded in source, never a custom parameter:

```javascript
import fs from 'fs';

export default async function run({ data }) {
  // 1) Get the temporary file path from Node 2 output
  const contentFilePath = data["{{2.result.file.content}}"];

  if (!contentFilePath) {
    throw new Error(
      'File path not found. Check that Node 2 outputs result.file.content and insert it via the helper widget.'
    );
  }

  // 2) Read file as a Buffer
  const contentFileBuffer = fs.readFileSync(contentFilePath);

  // 3) Modify (example: add ',"Processed"' column to each CSV row)
  const csvContent = contentFileBuffer.toString('utf8');
  const rows = csvContent.split('\n');

  const header = rows[0] ?? '';
  const processedRows = [header];

  for (let i = 1; i < rows.length; i++) {
    const row = rows[i];
    if (!row || row.trim() === '') continue;
    processedRows.push(`${row.trim()},"Processed"`);
  }

  const processedCsvString = processedRows.join('\n');
  const processedFileBuffer = Buffer.from(processedCsvString, 'utf8');

  // 4) Write and return as a file output
  const newFileName = 'processed_data.csv';
  fs.writeFileSync(newFileName, processedFileBuffer);

  return {
    file: file(newFileName),
    fileType: 'text/csv',
  };
}
```
#### Custom Parameter Types Reference
The JS code node supports custom parameters for dependency injection into your code.
The custom parameters schema is declared as a meta comment at the top of the code.
For example:
```javascript
/** @CustomParams
{
    "api_key": {
        "type": "string",
        "title": "API Key",
        "required": true
    },
    "endpoint": {
        "type": "string",
        "title": "API Endpoint",
        "required": true
    }
}
*/
```
In this example, the custom parameters are accessed via `data.api_key` and `data.endpoint` in your code.
Do **not** declare a custom parameter for a file. There is no CustomParams type that can accept file content. Hardcode `data["{{$3.result.fileInfo.content}}"]` (or that node's real `...content` path) in the source.
Important: `@CustomParams` is usually at the top of the code. A live Gmail+attachment example used `import` first, then `@CustomParams`, then the rest, and ran successfully.
##### JS code writing strategy
- First, write the JS code and describe your custom parameters schema inside your code.
- Then set the code parameter's value directly in the node's `parameters` object via `create_scenario` (new scenario) or `update_scenario` (existing scenario).
- Confirm the parameter's exact key and current schema via the `params` array returned by `search_node_types` for this node's alias.
- Fill custom parameter values as usual parameter values.
  Remember: when you change the `@CustomParams` schema you will be provided with a new parameters schema and values based on the new schema.
##### CONNECTION
For API tokens, keys, authorization data (object with an `access_token` or `api_key` field):
```json
"api_connection": {
"type": "connection",
"title": "API Connection",
"required": true,
"description": "Provide API credentials"
}
```
Usage: `data.api_connection.access_token` or `data.api_connection.api_key`.
##### STRING
Text input, optionally with a minimum length:
```json
"username": {
"type": "string",
"title": "Username",
"required": true,
"description": "Enter your username",
"options": {
"minLength": 3
    }
}
```
##### INT
Numeric input with optional min/max:
```json
"count": {
"type": "int",
"title": "Item Count",
"required": true,
"description": "Number of items to process",
"options": {
"min": 1,
"max": 100
    }
}
```
##### STRING_ARRAY
List of strings (tags, items, etc.):
```json
"tags": {
"type": "string_array",
"title": "Tags",
"required": false,
"description": "List of tags to apply",
"options": {
"maxCount": 10
    }
}
```
Usage: `data.tags` is an array: `data.tags.forEach(tag => ...)`.
##### STRING_TO_STRING
Key-value pairs (`Record<string, string>`):
```json
"settings": {
"type": "string_to_string",
"title": "Configuration",
"required": false,
"description": "Key:value pairs for configuration"
}
```
Usage: `Object.entries(data.settings).forEach(([key, val]) => ...)`.
##### SELECT
Dropdown with predefined options:
```json
"mode": {
"type": "select",
"title": "Operation Mode",
"required": true,
"description": "Select operation mode",
"options": {
"options": [
            { "key": "fast", "value": "Fast Mode" },
            { "key": "accurate", "value": "Accurate Mode" },
            { "key": "balanced", "value": "Balanced Mode" }
        ],
"multiple": false
    }
}
```
**CRITICAL**: SELECT value is ALWAYS `string[]` (array), even when `multiple: false`.
```javascript
// Correct - always access first element for single-select
const selectedMode = data.mode[0];  // "fast", "accurate", or "balanced"
```
**CRITICAL**: For float numbers use the `string` type and parse it to a float in your code.
#### End-to-end example
A single JS node that calls an API using an injected connection and returns parsed data. The `@CustomParams` block comes first (before imports), then the code:
```javascript
/** @CustomParams
{
    "api_connection": {
        "type": "connection",
        "title": "API Connection",
        "required": true,
        "description": "Provide API credentials"
    },
    "user_id": {
        "type": "string",
        "title": "User ID",
        "required": true,
        "description": "ID of the user to fetch"
    }
}
*/
import axios from "axios";
export default async function run({ data }) {
try {
const response = await axios({
method: "GET",
url: `https://api.example.com/users/${data.user_id}`,
headers: {
Authorization: `Bearer ${data.api_connection.access_token}`
            }
        });
return {
id: response.data.id,
name: response.data.name
        };
    } catch (error) {
return {
status: "error",
message: error.response?.data || error.message
        };
    }
}
```
Build it using the JS code writing strategy above (write code with its `@CustomParams` schema → set it in the node's `parameters` via `create_scenario`/`update_scenario` → confirm the schema via `search_node_types`).
#### Errors & debugging
- Use `console.log` for debugging — output appears in the node's Log tab. Never log secrets.
- For control flow: `throw` on unrecoverable errors, or return a structured error object (`{ status: "error", message }`) when the workflow should continue and inspect the error downstream.
- Large outputs may be truncated — return only the fields downstream nodes need.
#### Best practices checklist
**Shared (both runtimes)**:
- Always `export default async function run(...)` and always return an object (`{}` if nothing to return); never return primitives.
- Use `import`, never `require`; use `axios`, never `fetch`.
- Pin npm versions with `@` for reproducibility; rely on auto-install (never tell the user to install).
- Externalize user-controlled **non-file** configuration (API keys, emails, IDs) via `@CustomParams`. An upstream file is **only** `data["{{$3.result.fileInfo.content}}"]` (or that node's real `...content` path) hardcoded in the source — never a custom parameter. Decode with `Buffer.from(..., "latin1")`.
- Wrap I/O (HTTP, file, storage) in try/catch; return structured errors (`{ status: "error", message }`) or `throw` for unrecoverable cases.
- Keep within the 2-minute limit; use `Promise.all` for independent requests; split heavy work across multiple JS nodes.
- Use `console.log` for debugging (Log tab); never log secrets.
- Return only what downstream nodes need (keep payloads small; large outputs may be truncated).
- For files: write to the filesystem first, then wrap with the global `file()` / `files()` at the first level only.
  **Node.js (`use_bun: false`)**:
- Choose Node when a package or Node core module is unsupported or unstable on Bun.
- Full Node `fs` / core-module APIs are available for file handling.
  **Bun (`use_bun: true`, default)**:
- Default for speed; verify package compatibility (some libraries may behave differently — always check).
- Use Bun extras when helpful (e.g. built-in `bun:sqlite`); no server creation (`Bun.serve`) or ports.
- No TypeScript/JSX syntax — plain JavaScript only (applies to both runtimes, called out here since Bun users often expect TS).
