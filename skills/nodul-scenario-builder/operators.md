---
name: operators
description: Load when documentation of operator for templates is needed.
---

## Prefer operators over a JavaScript node
Before adding a JavaScript node, check whether the task can be done with template operators/functions. Operators are cheaper, faster (no cold start), and easier to maintain. For normal data iteration (running node chains per item in a list), prefer the **Iterator node** instead of a JS loop. Reach for a JS node only when the logic genuinely exceeds operator and Iterator capabilities — e.g. complex multi-step transforms, custom parsing beyond `split`/`replace`/regex, or calling libraries.
Quick "can operators do this?" checklist:

| Need                                  | Use                                                                             |
|---------------------------------------|---------------------------------------------------------------------------------|
| Conditionals                          | `if`, `switch`, `ifEmpty`                                                       |
| String work                           | `replace`, `split`, `substring`, `toLower`, `toUpper`, `trim`, `matchesPattern` |
| Arrays                                | `map`, `sort`, `distinct`, `merge`, `slice`, `join`                             |
| Iterating a list (run nodes per item) | Iterator node                                                                   |
| Math                                  | arithmetic (`+ - * / mod`), `sum`, `average`, `min`, `max`, `round`             |
| Dates                                 | `parseDate`, `formatDate`, `addMinutes`/`addHours`/`addDays`/`addMonths`        |
| Hashing / encoding                    | `md5`, `sha1`, `sha256`, `sha512`, `base64`, `encodeURL`                        |
---
## Identifiers (constants)
| Identifier  | Type      | Value                              |
|-------------|-----------|-------------------------------------|
| `null`      | null      | null                                |
| `true`      | bool      | true                                |
| `false`     | bool      | false                               |
| `now`       | date      | current UTC time in RFC3339 format  |
| `timestamp` | timestamp | current time as unix timestamp      |
| `space`     | string    | a single space character `" "`      |
### Data access syntax

These are the **only** prefixes for variables. `$` is always a **canvas node number** (`$1`, `$2`), never a node name.

| Prefix | Meaning | Write this | Never this |
|---|---|---|---|
| `$N` | Node output | `{{$1.data}}` | `{{$iterator_0.value}}` |
| `_` | Scenario / exec variables | `{{_.data}}` | |
| `%` | Global variables | `{{%.data}}` | |

Rare exception: `{{1.data}}` without `$`. `get_scenario` may *display* `$iterator_0`; convert to `$N` before any write.

Connection field access uses `#connId` (not a node variable). User-file access uses `^`. Neither replaces `$N`.

| Prefix | Description |
|---|---|
| ``$N.`field` `` | Node output (e.g. ``$1.`body`.`id` ``) |
| ``_.`field` `` | Execution context variables |
| ``%.`field` `` | Global variables |
| ``#connId.`field` `` | Connection data |
| ``^.fileId.`field` `` | User file access |
---
## Operators
Use parentheses `()` to group sub-expressions for logical and mathematical isolation / precedence, e.g. ``{{ ($1.`a` + $1.`b`) * 2 }}`` or ``{{ ($1.`x` > 0) and ($1.`y` > 0) }}``.
### Comparison (return `bool`)
| Operator     | Description           | Argument types           |
|--------------|-----------------------|--------------------------|
| `==` or `=`  | Equal                 | number, string, date     |
| `!=`         | Not equal             | number, string, date     |
| `>`          | Greater than          | number, string, date     |
| `>=`         | Greater or equal      | number, string, date     |
| `<`          | Less than             | number, string, date     |
| `<=`         | Less or equal         | number, string, date     |
String comparisons are lexicographic and **case-sensitive**: comparison is character-by-character, a lowercase letter is treated as greater than its uppercase counterpart, and the presence of a character is greater than its absence (so `"AB" > "A"`).
### Arithmetic
| Operator | Description                                                                 | Arguments                         |
|----------|-----------------------------------------------------------------------------|-----------------------------------|
| `+`      | Addition or string concatenation                                            | (number, number) or (string, any) |
| `-`      | Subtraction. With a single operand, negates it (unary minus). If both args are date — returns difference in milliseconds | (number) or (number, number) or (date, date) |
| `*`      | Multiplication                                                              | (number, number)                  |
| `/`      | Division                                                                    | (number, number)                  |
| `mod`    | Integer modulo (remainder)                                                  | (number, number)                  |
### Boolean
| Operator | Description     | Arguments    |
|----------|-----------------|--------------|
| `and`    | Logical AND     | (bool, bool) |
| `or`     | Logical OR      | (bool, bool) |
---
## Functions

IMPORTANT — case sensitivity: every function and operator name below is **camelCase and case-sensitive**. Writing a lowercased variant (e.g. `ifempty` instead of `ifEmpty`, `fromaiagent` instead of `fromAIAgent`) is a different, unrecognized identifier and will fail at runtime. Always copy the exact casing shown here.

Never invent an operator or function name. If the needed operation is not listed in this reference, use another documented composition or a JavaScript node.

### Arithmetic
| Function | Arguments | Returns |
|---|---|---|
| `ceil(x)` | number | number — rounds up |
| `floor(x)` | number | number — rounds down |
| `round(x)` | number | number — rounds to nearest |
| `parseNumber(str; decimalSep)` | string, string | number — parses number from string using a custom decimal separator |
| `formatNumber(num; precision[; decimalSep[; thousandsSep]])` | number, number[, string, string] | string — formats number. Default separators: decimal=`,`, thousands=`.`. Decimal and thousands separators must differ |
### Arrays
| Function | Arguments | Returns |
|---|---|---|
| `add(array; elem; ...)` | array, any (2+ args) | array — appends element(s) to array |
| `join(array; sep)` | array, string | string — joins array elements with separator |
| `slice(array; start[; end])` | array, number[, number] | array — subarray from start (exclusive) to end (inclusive). Pass empty instead of start to slice from 0 |
| `merge(array; array; ...)` | array (2+ args) | array — concatenates multiple arrays |
| `map(array; key[; filterKey; filterValues...])` | array, string[, string, any...] | array — extracts values by gjson path key; optional filter by another key |
| `sort(array[; order[; key]])` | array[, string[, string]] | array — sorts array. order values: `"asc"` (default), `"desc"`, `"asc ci"`, `"desc ci"` |
| `distinct(array; key)` | array, string | array — deduplicates by gjson path key |
| `deduplicate(array)` | array | array — removes duplicate values |
### Aggregate (accept array of numbers OR variadic numbers)
| Function | Arguments | Returns |
|---|---|---|
| `average(array)` or `average(n1; n2; ...)` | array or number (1+) | number |
| `min(array)` or `min(n1; n2; ...)` | array or number (1+) | number |
| `max(array)` or `max(n1; n2; ...)` | array or number (1+) | number |
| `sum(array)` or `sum(n1; n2; ...)` | array or number (1+) | number |
### Control flow
| Function | Arguments | Returns |
|---|---|---|
| `if(cond; thenVal; elseVal)` | bool, any, any | any |
| `ifEmpty(val; default)` | any, any | any — returns default if val is empty |
| `switch(val; case1; result1; ...[; default])` | any, (any, any)+, [any] | any — matches val against cases; optional trailing default |
### Boolean / logic
| Function | Arguments | Returns |
|---|---|---|
| `not(x)` | bool | bool |
| `empty(x)` | any | bool — true if string/array/object is empty, or if null. Numbers and bools are never empty |
| `contains(container; item)` | (string\|number, string\|number) or (array, any) | bool |
| `length(x)` | string, array, or object | number — character count / element count / key count |
### Strings
| Function | Arguments | Returns |
|---|---|---|
| `startsWith(str; prefix)` | string, string | bool |
| `endsWith(str; suffix)` | string, string | bool |
| `matchesPattern(str; regex)` | string, string | bool — regexp match |
| `toLower(str)` | string | string |
| `toUpper(str)` | string | string |
| `replace(str; old; new)` | string, string, string | string — replaces all occurrences |
| `trim(str)` | string | string — strips leading and trailing whitespace |
| `substring(str; start; end)` | string, number, number | string — unicode-aware slice from start (exclusive) to end (inclusive) |
| `indexOf(str; substr)` | string, string | number — unicode character position, or -1 if not found |
| `split(str; sep)` | string, string | array of string — each substring is trimmed; empty substrings are dropped |
| `encodeURL(str)` | string | string — URL query encode |
| `decodeURL(str)` | string | string — URL query decode |
| `escapeMarkdown(str)` | string | string — escapes Markdown special characters |
| `jsonStringify(val)` | any | string — JSON.stringify |
| `md5(str)` | string | string — hex hash |
| `sha1(str)` | string | string — hex hash |
| `sha256(str)` | string | string — hex hash |
| `sha512(str)` | string | string — hex hash |
| `base64(str)` | string | string — base64 encoded |

Template data-access expressions resolve objects/arrays as JSON automatically when the destination field accepts structured data. Use `jsonStringify` mainly when you deliberately need a JSON **string**, especially to escape a value inside manually composed JSON text; do not stringify an object merely to pass it between normal structured fields.
### Date / time
Date values are accepted in RFC3339 format or as a unix timestamp (number).
`formatDate` and `parseDate` use [gostradamus](https://github.com/bykof/gostradamus) format tokens.

| Function | Arguments | Returns |
|---|---|---|
| `addMinutes(date; n)` | date, number | date |
| `addHours(date; n)` | date, number | date |
| `addDays(date; n)` | date, number | date |
| `addMonths(date; n)` | date, number | date |
| `setMinute(date; n)` | date, number | date — sets minute to n |
| `setHour(date; n)` | date, number | date — sets hour to n |
| `setDay(date; n\|weekday)` | date, number or string (e.g. `"Monday"`) | date — sets day of week. n in 1-7 stays within the current Sun-Sat week (1=Sun … 7=Sat); values outside roll into the previous/next week |
| `formatDate(date; format[; timezone])` | date, string, string | string — formats date using gostradamus pattern |
| `parseDate(str; format[; timezone])` | string, string, string | date — parses date string. If first arg is a number/timestamp, format is not required |
### Data access
| Function | Arguments | Returns |
|---|---|---|
| `get(obj; path)` | object\|array, string | any — extracts value by gjson path |
| `getGlobalVar(key)` | string | any — gets global variable by key |
### Agent / MCP input (context-restricted)
| Function | Arguments | Returns |
|---|---|---|
| `fromAIAgent(name; description)` | string, string | any — value the AI Agent supplies at runtime. **Only** inside nodes connected to an AI Agent (Agent Tools). 1st arg = parameter name the agent sees, 2nd = description of what to pass. |
| `fromMCP(name; description)` | string, string | any — value the MCP caller supplies. **Only** inside MCP scenarios (require an MCP Trigger + MCP Response). Same two-field principle as `fromAIAgent`. |
Both are FORBIDDEN outside their context — calling them in any other node/scenario throws at runtime (e.g. "The fromAIAgent operator can only be specified within nodes connected to the AI Agent node.").
Tip: most fields that accept `fromAIAgent`/`fromMCP` show a **Let AI Decide** button that inserts the correct placeholder and auto-generates a parameter name + description for you.
### Built-in AI
| Function | Arguments | Returns |
|---|---|---|
| `AskAI(prompt)` | string | string — sends `prompt` to the built-in AI and returns its text response |
Unlike `fromAIAgent`/`fromMCP`, `AskAI` works **anywhere** — in normal fields and in route conditions. Embed variables/node outputs into the prompt with `+` concatenation, e.g. `{{ AskAI("Is \"" + _.Text + "\" a negative review?") }}`.
Caveat: responses are non-deterministic — verify outputs before relying on them. For use in a route condition, instruct the AI to reply with a single word and compare it, e.g. `{{ AskAI("... return one word \"true\" or \"false\"") = "true" }}`.
---
## Routes & filters
- A route's Condition must evaluate to a boolean (`true`/`false`). If `true`, execution continues through that route; if `false`, it does not.
- A **fallback route** fires only when none of the node's outgoing routes evaluate to `true`.
- To debug a filter, copy the same expression into a Set Variables node in a neighboring branch — its output shows the input values and the final `true`/`false` result.
---
When you are using concatenation of strings inside function arguments you must use + operator. For example: {{ $1.`body`.`id` + "_" + $1.`body`.`name` }}
When you type 2 independent templates there is no need to use + operator. For example: {{ $1.`body`.`id` }}_{{ $1.`body`.`name` }}
