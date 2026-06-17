---
name: ai_agent_node
description: Load when working with ai_agent node.
---

### AI Agent node
The AI Agent is a special action node that implements the classic LLM **function-calling / tool-calling** pattern. Each time it runs it reads its instructions (system message), analyzes the input, and decides whether to reply directly or call one of its connected tools. It can call tools repeatedly, looping until it reaches a finished state, bounded by `Max Iterations`.
#### When to use the AI Agent node
Use the AI Agent node ONLY when you genuinely need **non-deterministic, multi-step logic** — i.e. the model itself must decide which tool(s) to call and how many times, based on the input. It is more expensive (tokens, latency) and harder to make predictable.
For **deterministic, single-turn** LLM work — summarization, sentiment/classification, extraction, translation, straightforward text generation — do NOT use an AI Agent. Use the simpler **AI nodes** instead (e.g. **AI GPT Router** / **All LLM Models**), which are a direct LLM call with no autonomous tool loop.
Rule of thumb: use the AI Agent node only if you truly need non-deterministic, tool-using logic; for simple cases use the AI nodes (AI GPT Router / All LLM Models).
#### Rules at a glance
- **Pick the right node.** AI Agent only for non-deterministic, multi-step, tool-calling logic; for simple/single-turn tasks use the AI nodes (AI GPT Router / All LLM Models).
- **Ambiguity rule.** If you are unsure whether a node should be a **tool** of the agent or a **sequential continuation**, STOP and ask the user. Do not guess — present both options briefly and let the user decide.
- **Unique names.** Every node must have a unique, descriptive name/alias. A missing or duplicate name fails the run.
- **Tool Description.** Give each tool a **short but descriptive enough**, action-oriented description (e.g. "fetch current weather data"). Vague or overlong descriptions get the tool ignored or misused.
- **Params are strings.** `fromAIAgent()` values are always strings. No binary/file data through tools.
- **Isolation.** Tools cannot read each other's outputs; nodes placed after the agent cannot read tool outputs.
- **Iterations.** `Max Iterations` bounds tool calls per run; when reached, the agent halts and replies that it stopped.
#### AI Agent node fields
Configure these inside the AI Agent node's parameters. Keys confirmed from the example below are `model`, `system_message`, `user_prompt`, `max_tokens`, `temperature`; for the other fields, read the exact `paramValues` key with `get_workflow_node_state` before writing.
- **Model** (`model`) — the LLM, in `provider/model` form. **Default to `anthropic/claude-sonnet-4.6`** unless the user asks for a different model. Enumerate available models with the **List Models node**.
- **System Message** (`system_message`) — the agent's instructions: role, allowed/disallowed actions, when to call each tool.
- **User Prompt** (`user_prompt`) — the main input/query. Supports variable interpolation.
- **Temperature** (`temperature`) — sampling temperature (0–2). Lower = more deterministic.
- **Max Tokens** (`max_tokens`) — upper bound on generated tokens.
- **Session ID** — identifier (e.g., chat id or user id) used to load/separate short-term conversation memory. Empty = stateless, treated as a new session.
- **Context Window Length** — number of recent message pairs included in the context.
- **Max Iterations** — cap on tool invocations in a single reasoning run; on reaching it the agent stops and reports it was halted.
- **Structured output** (switch) + **Output JSON Schema** — when enabled, forces the response into JSON matching the provided schema.
  The agent's reply is exposed on its output as `text` (read it downstream with `{{ $<agentNodeName>.text }}`); with Structured output enabled it also returns the parsed JSON object.
#### How tools work
All tools defined in the agent's `tools` parameter. A tool can be a **single action node** or a **chain of action nodes** (route from the agent to the first node, then connect the rest in sequence; invoking the tool runs the whole chain in order).
What the LLM sees for each tool:
- **Name** — the name (alias) of the **first** node in the chain.
- **Description** — a separate **Tool Description** field on that first node (written via `paramValues`, shown as `tool_description` in the example below; confirm the exact key via `get_workflow_node_state`).
- **Arguments** — collected by scanning every `{{ fromAIAgent() }}` expression across all node parameters in the tool's chain.
  Returning a result to the agent:
- Place an **AI Agent Tool Response** node at the **end** of the chain to define exactly what is returned to the agent (confirm its exact node alias/type via `get_workflow_node_state` on an existing one).
- If there is no AI Agent Tool Response node, the **last node's output** in the chain is returned to the agent.
- Use this to trim large payloads (e.g., a node returning big JSON) down to compact text before it re-enters the agent's context, reducing token use and context-limit errors.
  Data-flow rules:
- Tool nodes can read data from previous nodes within their own chain and from nodes upstream of the AI Agent.
- One tool's chain cannot read another tool's chain — tools cannot read each other's outputs.
- Nodes placed AFTER the AI Agent cannot read data from its tool nodes.
#### `fromAIAgent()` syntax
Use `fromAIAgent()` inside any input field of a tool node (URL, Request Body, Prompt, Text, headers, etc.) as a placeholder the agent fills at run time:
```
{{ fromAIAgent("paramName"; "description") }}
```
- Arguments are separated by a **semicolon** (`;`), not a comma.
- First argument: the parameter name on the tool node.
- Second argument: the parameter description — optional, but shown to the model and used in the function schema. Prefer including it.
- The value is always a **string**.
#### Example: webhook → agent with HTTP tool
A workflow triggered by a webhook with an AI Agent that can perform HTTP requests.
NOTE: the `paramValues` shown inside each node below are INLINED for illustration only — to make the example self-contained on one page.
```
{
  "name": "Test AiAgent",
  "description": "A test scenario for demonstration purposes",
  "nodes": [
    {
      "name": "Webhook",
      "typeAlias": "webhook",
      "paramValues": {
        "path": "c7a98c62-0aaf-415e-85bf-736b1ecb4a96"
      }
    },
    {
      "name": "AiAgent",
      "typeAlias": "ai_agent",
      "paramValues": {
        "model": "anthropic/claude-sonnet-4.6",
        "system_message": "You are a helpful assistant that can do http requests.",
        "user_prompt": "{{ $Webhook.`body`.`message` }}",
        "max_tokens": 1000,
        "temperature": 0.5,
        "tools": [
          {
            "name": "HttpRequest",
            "typeAlias": "http_request",
            "paramValues": {
              "tool_description": "Send an HTTP request to a URL and return the response.",
              "url": "{{ fromAIAgent(\"url\") }}",
              "method": "{{ fromAIAgent(\"method\"; \"GET or POST\") }}",
              "headers": {
                "Content-Type": "{{ fromAIAgent(\"contentType\"; \"for example: application/json\") }}"
              },
              "body": "{{ fromAIAgent(\"body\"; \"leave empty if you don't need to send a body\") }}"
            }
          },
        ]
      }
    },
    {
      "name": "RespondToWebhook",
      "alias": "respond_to_webhook",
      "paramValues": {
        "status": "200",
        "headers": {
          "Content-Type": "application/json"
        },
        "body": "{\"agentResponse\": \"{{ $AiAgent.`text` }}\"}"
      },
    }
  ]
}
```
Here the workflow starts from the `webhook` trigger and runs the `ai_agent` node.
If the webhook's message asks the agent to make an HTTP request, the workflow is rerouted into the `http_request` node, which executes.
After `http_request` completes, control returns to `ai_agent`, which runs again with the tool-call result.
Once the AI Agent reaches its finished state, the workflow continues as usual in the graph — so `respond_to_webhook` is executed with the AI Agent's result.
Under the hood, the AI Agent and the HTTP request node combine into the classic tool-calling pattern: the agent has a single tool `http_request` (described by its Tool Description) with parameters `url`, `method`, `contentType`, `body`.
#### Authoring the System Message & Tool Descriptions
Models retain the start and end of the prompt most strongly, so order the System Message:
1. **Role** — who the agent is (2–4 sentences).
2. **Tools** — which tools exist, when to call each, in what order. Names in the prompt must match the node names exactly.
3. **Goal** — what counts as success in one interaction.
4. **Environment** — channel/context and what the agent cannot see.
5. **Tone** — brevity, formality, acknowledgements.
6. **Guardrails** — boundaries, refusals, escalation.
7. **Closing reminder** (optional) — repeat the single most important rule.
   Guidelines:
- Use descriptive tool names (`create_invoice`, `send_email`) — never `Node 3` / `doStuff`.
- Tool Descriptions: short but descriptive enough, action-oriented, one line saying what the tool does and when to call it. Not vague, not a wall of text.
- Validate required inputs before acting; confirm intent before irreversible actions (sending email, booking, payments); return structured errors the agent can interpret (e.g. `{"status":"error","message":"Missing required field: email"}`).
- Connect only the tools the agent needs; design each tool to do one task well.
- For structured responses, enable Structured output and define an Output JSON Schema.
#### Think & Plan tools (don't use by default unless the user asks for it)
Optional reasoning tools that improve accuracy at the cost of tokens/latency:
- **Think Tool** — forces the agent to write reasoning between steps; with `Force Think Tool` it is injected on every step. Best when correctness/traceability matter most.
- **Plan Tool** — generates one structured plan up front, then follows it. Cheaper/faster than Think, less consistent. Best for medium-complexity or repetitive flows.
#### Multi-agent workflows (don't use by default unless the user asks for it)
For complex tasks, an AI Agent can itself be a tool of another AI Agent, splitting responsibilities across specialized agents (e.g. copywriter, editor, fact-checker). This reduces hallucinations from overloading one agent and suits multi-step reasoning, tool orchestration, and pipelines that need clear logic separation.
#### Constraints & limitations
- **Data flow.** Tool nodes can read previous nodes in their own chain and nodes upstream of the agent. Tools cannot read each other's chains, and nodes after the agent cannot read tool outputs.
- **No binaries.** AI Agents work only with text. You cannot pass files between tools or return a file produced inside a tool. Even a text file returned as a file object cannot be consumed by the agent — return text-file contents as a string and use them in code if needed.
- **Strings only.** Every `fromAIAgent()` parameter is a string.
- **Iterations.** Tool calls per run are bounded by `Max Iterations`; on reaching it the agent stops and reports it was halted.
