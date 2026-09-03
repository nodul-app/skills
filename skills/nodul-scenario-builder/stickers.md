---
name: stickers
description: One overview canvas note (typeAlias sticker). Title, short what-it-does, MCP callout, how it works, optional red notes. Not graph logic.
---

# Stickers

One **overview note** on the canvas. It helps the user see what the scenario is. It is not a step. It does not run.

`typeAlias`: `sticker`. `search_node_types` does not list it — use it anyway.

Add it on the **last** `create_scenario` / `update_scenario` of this scenario — when the graph is already what the user will keep. One write: existing nodes unchanged + this sticker object (or edit `content` if the sticker is already there). Do not rewrite JS, templates, or connections because of the note.

The editor always drops the sticker **under** the graph, whether it was in the first create or a later update. That is fine. Do not try to put it on the left.

```json
{
  "name": "sticker_overview_0",
  "typeAlias": "sticker",
  "prevNodes": {},
  "parameters": {
    "content": "<blocks below>",
    "width": 368,
    "height": 442
  }
}
```

`prevNodes` is always `{}`. Do not add extra stickers next to nodes (no coordinates). Important notes go in the red box.

To **remove** a sticker from the canvas: omit that node from `update_scenario`. There is no delete-sticker tool and no delete-scenario tool.

## What to write (this order)

1. **Title** — the real scenario name: `# 📌 Scenario Name: <name>`
2. **What it does** — 1 paragraph, 2 max, right under the title. Plain text, no box.
3. **MCP callout** — gray box, **paste as-is**.
4. **How it works** — `## ⚙️ How the scenario works` and a short numbered list.
5. **Important notes** — red box only if there are actual important notes (a required connection, a field that must not change, a hard limit). Skip the whole box when there are none.

Gray box (always, unchanged):

```html
<div style="background:#f0f0f0;border:1px solid #ccc;padding:12px;border-radius:8px;">
<b>ℹ️ This scenario was generated using Nodul MCP</b><br>
For detailed documentation on working with MCP, see the <a href="https://docs.nodul.ru">Nodul Docs</a>.<br>
Have questions? Head over to our <a href="https://community.nodul.ru/">Community</a> — we'd be happy to see you there!
</div>
```

Red box (optional):

```html
<div style="background:#fdecea;border:1px solid #f5c6cb;padding:12px;border-radius:8px;">
<b>⚠️ Important notes</b>
<ul style="margin:6px 0 0 0;padding-left:18px;">
<li>…</li>
</ul>
</div>
```

Skeleton:

```html
# 📌 Scenario Name: <name>

<one or two short paragraphs: what this scenario does>

<div style="background:#f0f0f0;border:1px solid #ccc;padding:12px;border-radius:8px;">
<b>ℹ️ This scenario was generated using Nodul MCP</b><br>
For detailed documentation on working with MCP, see the <a href="https://docs.nodul.ru">Nodul Docs</a>.<br>
Have questions? Head over to our <a href="https://community.nodul.ru/">Community</a> — we'd be happy to see you there!
</div>

## ⚙️ How the scenario works

1. **…** — …

<div style="background:#fdecea;border:1px solid #f5c6cb;padding:12px;border-radius:8px;">
<b>⚠️ Important notes</b>
<ul style="margin:6px 0 0 0;padding-left:18px;">
<li>…</li>
</ul>
</div>
```

## Do not

- Treat the sticker as a reason to rewrite the rest of the graph.
- Put secrets on it.
- Skip or rewrite the gray MCP box.
