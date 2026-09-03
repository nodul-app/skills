---
name: working_with_files
description: Load when working with file inputs/outputs across nodes (webhook response, upload, JS node file paths, Gmail attachments, binary content paths).
---

### Working with Files

When configuring nodes that work with files (sending, uploading, attaching, returning, processing), follow these rules.

#### Binary files — path MUST end with a leaf field (CRITICAL)

For a binary file, never pass a parent object (`file`, `fileInfo`, `files`, `output`). The template **must contain and end with** one of:

| Leaf | What it is | Typical template |
|---|---|---|
| `content` | The file bytes (what you put in attach/upload/JS file inputs) | `{{$3.body.file.content}}` |
| `extension` | File extension | `{{$3.body.file.extension}}` |
| `filename` | File name including extension | `{{$3.body.file.filename}}` |

**Most of the time the path ends with `content`.** That is the value for attachment/upload fields.

Use the upstream node's **canvas number**, not its `name` and **not** its position in `get_scenario` `nodes[]`. After a delete, a JS `file()` node may be `$7` even if it is listed second. If you remove a producer, rewrite every downstream `...content` path (Gmail `attachments`, JS `data["{{$N...}}"]`).

```text
{{$7.file.content}}
```

`get_scenario` may **display** `$3` as a node name (for example `$image_0.result.fileInfo.content`). That display is **not** valid to write back. Always send `{{$N....content}}` (digits after `$`).

Do **not** write `{{$image_0.result.fileInfo}}` or `{{$3.result.fileInfo}}`. Those stop on a container. The engine does not treat that as a file.

##### Middle segments are node-specific — do not invent them

The path **before** `content` / `extension` / `filename` comes from **that node's actual output**. Copy it from the live output (or from `get_execution` / the data picker). Do not reuse a segment from another node type.

| Example path | Where it comes from |
|---|---|
| `{{$3.body.file.content}}` | **Usual** shape: `body.file.content` |
| `{{$3.result.fileInfo.content}}` | **This path is not universal.** Image Generation (`fileInfo`) is one node that happens to nest the file under `result.fileInfo`. Other nodes will not have `fileInfo`. |
| `{{$3.result.fileInfo.extension}}` | Same node: extension only |
| `{{$3.result.fileInfo.filename}}` | Same node: filename only |

If the output is an **array** of files, still end on `content` (or `extension` / `filename`), with an index:

```text
{{$3.body.files.[0].content}}
```

Wrong:

```text
{{$3.body.file}}
{{$3.result.fileInfo}}
{{$image_0.result.fileInfo}}
```

Right (bytes):

```text
{{$3.body.file.content}}
{{$3.result.fileInfo.content}}
```

#### Gmail Send Email — `attachments`

`attachments` is `string_to_string` (key + value). This is not special-cased: the **value** is still a file **content** path.

- **Key** — filename including extension, e.g. `final-test-sunset.webp`
- **Value** — the binary content template, ending with `content`

```json
"attachments": {
  "final-test-sunset.webp": "{{$3.result.fileInfo.content}}"
}
```

For a typical webhook/file node the value would be `{{$3.body.file.content}}`. The Gmail field description also allows a public file URL in the value; for a file produced earlier in the scenario, use the `content` leaf, not the parent object.

#### What a file object looks like in output

Live output may look like this (field names and nesting **vary by node**):

```json
{
  "filename": "9044275.png",
  "size": 3658,
  "content": "iVBORw0KGgoAAAANSUhEUgAAAYIAAAGCCAMAAADaJDS7AAAA1VBMVEUAAABgv69Yv69Yv69cv69cv6xdv61cv61dv61bv69cv61cv65dv61dv61cv6xdv61gv69dv61cv69gvw==",
  "headers": {
    "Content-Disposition": ["form-data; name=\"file\"; filename=\"9044275.png\""],
    "Content-Type": ["image/png"]
  },
  "extension": ".png",
  "mediatype": "image/png",
  "path": ""
}
```

Use `.content` / `.extension` / `.filename` on whatever object actually contains those keys. Do not pass the object itself into a file parameter.

#### Common use cases

1. **Returning a file in a Webhook Response** (`respond_to_webhook`):
   - Set appropriate headers (e.g., `Content-Type`, `Content-Disposition`).
   - **When setting headers** (Content-Type or any other header): first check if this header already exists with a different value. This ensures only one value per header exists (following the `string_to_string` atomic operation pattern).
   - In the body, use the **content** path, e.g. `{{$3.body.file.content}}`.

2. **Uploading or attaching a file** (Gmail, Drive, HTTP file fields, etc.):
   - Value = `{{$N....content}}` (path ends with `content`).
   - If the target also needs a name, put `filename` in the filename/key field, not in the content value.

3. **Passing files between nodes**:
   - Same rule: destination file inputs get `content`, not the parent.

4. **JavaScript nodes (STRICT):**
   - There is **no** CustomParams type for file content. A custom string/connection field **cannot** take `{{$3.result.fileInfo.content}}` (or any `...content` path) and pass it into `data`. That breaks the node.
   - Hardcode the accessor **in the source**, this format (middle segments from the live output; leaf always `content`):

```javascript
const webpContent = data["{{$3.result.fileInfo.content}}"];
const imageBuffer = Buffer.from(webpContent, "latin1");
```

   - Or, when the accessor is a **temp path**: `const contentFilePath = data["{{2.result.file.content}}"];` then `fs.readFileSync(contentFilePath)`, then write + `file()`.
   - Equivalent numbered form: `data["{{3.`result`.`fileInfo`.`content`}}"]`.
   - Typical other nodes: `data["{{$3.body.file.content}}"]`.
   - After a JS node `return { file: file("name.csv") }`, the next JS reads `data["{{$2.file.content}}"]` (no `result` wrapper).
`@CustomParams` is fine for text, recipient, Gmail connection — **never** for a file. Do **not** put `{{$3.result.fileInfo.content}}` (or any `...file...content` path) in `parameters` next to `code`. That is a CustomParams *value* and it breaks. The file exists only as `data["{{3.`result`.`fileInfo`.`content`}}"]` inside the source.

#### Files in AI Agent tool nodes (IMPORTANT)

AI Agents cannot work with binary data. You cannot pass files between tools, and you cannot return a file produced inside a tool. AI Agents work only with text data. Even if a tool returns a text file as a file object (rather than a string), the AI Agent cannot consume it. Return text-file contents as a string and use them in your code if needed.

#### Reading an upstream file inside the JavaScript node (official pattern)

**Only** a hardcoded `data["{{$N....content}}"]` in the source. Never a custom parameter.

1. **Read bytes** — accessor must end with `content`. Decode with `latin1` or the file is corrupted:

```javascript
const webpContent = data["{{$3.result.fileInfo.content}}"];
if (!webpContent) {
  throw new Error("File content not found.");
}
const imageBuffer = Buffer.from(webpContent, "latin1");
```

2. If you need to **return** a file to downstream no-code nodes: write with `fs.writeFileSync`, then wrap with `file(...)` at the first level.

The same hardcoded accessor is also used when the platform gives a **temporary filesystem path** (official Handling Files example). Then read with `fs.readFileSync`, modify, write a new file, return `file(...)`. Do not put the path in CustomParams.

```javascript
import fs from 'fs';

export default async function run({ data }) {
  // 1) Temporary file path from Node 2 — hardcoded, ends with content
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

Two ways to consume `data["{{...content}}"]` after the hardcoded lookup:

| What the accessor returns | How to turn it into bytes |
|---|---|
| Binary string (e.g. Image Generation `fileInfo.content`) | `Buffer.from(value, "latin1")` |
| Temporary filesystem path (this CSV / Handling Files example) | `fs.readFileSync(contentFilePath)` |

Gmail send from JS (Image Generation file + connection custom param, **no file custom param**):

```javascript
import fs from "fs";
import axios from "axios";

/** @CustomParams
{
  "email_body": {
    "type": "string",
    "title": "Email body",
    "required": true
  },
  "recipient": {
    "type": "string",
    "title": "Recipient email",
    "required": true
  },
  "gmail_connection": {
    "type": "connection",
    "title": "Gmail connection",
    "required": true
  }
}
*/

const ATTACHMENT_FILENAME = "final-test-sunset.webp";
const ATTACHMENT_MIMETYPE = "image/webp";

function wrapBase64(value) {
  const lines = [];
  for (let i = 0; i < value.length; i += 76) {
    lines.push(value.slice(i, i + 76));
  }
  return lines.join("\r\n");
}

function getToken(conn) {
  if (!conn) return "";
  let value = conn;
  if (typeof conn === "string") {
    try { value = JSON.parse(conn); } catch (e) { return conn; }
  }
  if (typeof value === "object" && value) {
    return value.access_token || value.api_key || value.token || "";
  }
  return "";
}

export default async function run({ data }) {
  const emailBody = data.email_body || "";
  const recipient = data.recipient || "";
  const token = getToken(data.gmail_connection);

  const webpContent = data["{{3.`result`.`fileInfo`.`content`}}"];
  if (!webpContent) {
    throw new Error("WebP file content not found.");
  }
  const imageBuffer = Buffer.from(webpContent, "latin1");

  const boundary = "nodul_mime_boundary";
  const imageB64 = wrapBase64(imageBuffer.toString("base64"));

  const mime = [
    `To: ${recipient}`,
    `Subject: Nodul file attachment`,
    "MIME-Version: 1.0",
    `Content-Type: multipart/mixed; boundary="${boundary}"`,
    "",
    `--${boundary}`,
    "Content-Type: text/plain; charset=UTF-8",
    "Content-Transfer-Encoding: 7bit",
    "",
    emailBody,
    "",
    `--${boundary}`,
    `Content-Type: ${ATTACHMENT_MIMETYPE}; name="${ATTACHMENT_FILENAME}"`,
    `Content-Disposition: attachment; filename="${ATTACHMENT_FILENAME}"`,
    "Content-Transfer-Encoding: base64",
    "",
    imageB64,
    `--${boundary}--`,
    ""
  ].join("\r\n");

  const raw = Buffer.from(mime, "utf8")
    .toString("base64")
    .replace(/\+/g, "-")
    .replace(/\//g, "_")
    .replace(/=+$/, "");

  const response = await axios.post(
    "https://gmail.googleapis.com/gmail/v1/users/me/messages/send",
    { raw },
    {
      headers: {
        Authorization: `Bearer ${token}`,
        "Content-Type": "application/json"
      }
    }
  );

  return {
    gmail_message_id: response.data.id,
    threadId: response.data.threadId,
    to: recipient,
    attachment_filename: ATTACHMENT_FILENAME,
    attachment_size: imageBuffer.length
  };
}
```
