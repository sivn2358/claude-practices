# CCA Domain 2 Cheat Sheet — Tool Design & MCP Integration

Nov 2, 2026 · @Si

## At a glance

Domain 2 is **18%** of the exam. It tests three things: whether Claude picks the right tool, whether a tool's error tells the agent what to do next, and whether each agent has only the tools its role needs.

| # | Task statement | Section below |
| --- | --- | --- |
| 2.1 | Design tool interfaces with clear descriptions and boundaries | Tool descriptions |
| 2.2 | Implement structured error responses for MCP tools | MCP errors |
| 2.3 | Distribute tools across agents and configure tool choice | Tool scoping and tool_choice |

**The five rules that answer most questions**

1. Claude picks tools by their **descriptions**. If it calls the wrong tool, fix the descriptions first.
2. A good description says what the tool does, its input format, what it returns (including an empty result), when to use it, and when **not** to use it.
3. **An empty result is not an error.** A failed execution returns `isError: true` with a category and a retryable flag.
4. Retry only `transient` errors. For `validation`, fix the input. For `business`, escalate. For `permission`, fix credentials or escalate.
5. Least privilege: give each agent 4–5 tools that fit its role. Force a specific tool with `tool_choice` for fixed pipeline steps.

## Tool descriptions (2.1)

Claude has no routing layer other than tool names and descriptions. Vague descriptions cause it to call the wrong tool. Overlapping descriptions make its choice unpredictable. Clearly different descriptions make routing reliable.

**What a good description covers**

| Part | Example |
| --- | --- |
| What it does, specifically | "Searches internal policies and product docs" |
| Input format | "customer_id like C-12345", "ISO 8601 date range" |
| What it returns | "Up to 10 articles: id, title, snippet" |
| Empty-result meaning | "Returns [] when nothing matches. This is a normal result, not an error." |
| When to use / when not | "Internal info only. For current events use search_web. For customer data use lookup_customer." |

```json
{
  "name": "search_knowledge_base",
  "description": "Searches the internal knowledge base for company policies, internal procedures and product documentation. Input: a natural-language query. Returns up to 10 articles with id, title and snippet. Returns an empty array when nothing matches; this is a normal result, not an error. Use for internal information only. Do not use it for current events or public facts (use search_web) or for customer-specific data (use lookup_customer)."
}
```

**When to split a tool**

- A `type` or mode parameter changes its behavior completely.
- Its description needs several "if X, then Y" branches.
- Different callers need different parts of it.
- It is called wrongly more than about 5% of the time.

Example: an overloaded `analyze_document` tool with web, PDF and database modes becomes `extract_web_results`, `parse_pdf_sections` and `query_database_record`.

**System prompt conflicts.** Wording in the system prompt can pull Claude toward a tool with a similar name. For example, "analyze content from each URL" can trigger `analyze_content`. Check system prompts for phrases that match tool names.

## MCP errors (2.2)

**Fields in an error response**

| Field | Meaning |
| --- | --- |
| `isError` | `true` |
| `errorCategory` | `transient`, `validation`, `business` or `permission` |
| `isRetryable` | boolean |
| `message` | Specific and readable: what failed and why |
| `retryAfterMs` | Optional backoff hint |

**Categories**

| Category | Example | Retryable | Agent action |
| --- | --- | --- | --- |
| `transient` | DB timeout, service down | Yes | Retry with exponential backoff |
| `validation` | Wrong ID format, missing field | No | Fix the input, then call again |
| `business` | Refund over limit | No | Escalate or tell the user |
| `permission` | Access denied, wrong scope | No | Rotate credentials or escalate |

**Empty result vs error**

| Situation | Response | Agent does |
| --- | --- | --- |
| Query worked, nothing matched | `isError: false`, empty content, info message | Accept it and continue |
| Execution failed | `isError: true`, category, retryable, message | Handle it by category |

If empty results are treated as errors, the agent retries successful queries forever. If errors are treated as empty results, data is silently lost.

```python
# No match: this is a successful query
return {"isError": False,
        "content": [{"type": "text", "text": "No orders found for C-1042"}]}

# Timeout: a retryable error
return {"isError": True,
        "content": [{"type": "text", "text": json.dumps({
            "errorCategory": "transient", "isRetryable": True,
            "retryAfterMs": 2000, "message": "Order DB timed out after 5s"})}]}
```

**Agent-side handling, in order**

1. If `isError` is false, process the content. Empty content is fine.
2. If the error is retryable and retries remain, wait `retryAfterMs` (default about 1000 ms), then retry.
3. If the category is `business`, escalate to a human with the message.
4. If the category is `permission`, alert operations.
5. Otherwise, log the error and pass it up to the coordinator.

## Tool scoping and tool_choice (2.3)

**Least privilege.** Give each agent only the tools its role needs, about 4–5. With fewer tools, each description stands out and selection is more reliable. Tools outside an agent's role get misused, for example a synthesis agent that searches the web instead of synthesizing.

| Agent | Tools |
| --- | --- |
| Coordinator | Task/Agent (spawn), search_web |
| Web search | search_web, fetch_page |
| Document analysis | read_file, extract_document_sections |
| Synthesis | summarize_findings, format_citations (**no web search**) |
| Report | format_report, write_file |

**Tool profiles by task phase**

| Phase | Tools | Permission mode |
| --- | --- | --- |
| Exploration | Read, Glob, Grep | default |
| Development | + Write, Bash | acceptEdits |
| Deployment | + run tests, deploy to staging, notify | reviewed |

**`tool_choice`**

| Value | Behavior | Use when |
| --- | --- | --- |
| `auto` (default) | Claude decides whether to call a tool | Normal agents |
| `any` | Must call some tool, picks which | A tool call is required, any of several fits |
| `{"type": "tool", "name": ...}` | Must call that tool | A fixed pipeline step, e.g. always `extract_structured_data` |

**`strict: true`** on a tool definition makes Claude's tool inputs match the schema: types, required fields and enums.

## Anti-patterns and exam triggers

Most wrong answers swap a deterministic mechanism for a probabilistic one, or a precise signal for a vague one.

| If the question says… | The answer is… | Trap answer |
| --- | --- | --- |
| Claude keeps calling the wrong tool | Rewrite descriptions: input, output, when to use, when not | Add more tools, or say "use the right tool" in the prompt |
| Two tools have near-identical descriptions | Make them clearly different, or split by source | Rename them only |
| Query found no records | `isError: false` + empty content | Return an error, so the agent retries forever |
| Subagent's tool timed out | Structured error: `transient`, `isRetryable: true` | Return empty results (silent data loss) |
| Agent retries a bad-format request | `validation` error, not retryable, fix input | Keep retrying with backoff |
| Refund exceeds policy | `business` error → escalate | Retry |
| Synthesis agent is browsing the web | Remove web search from its tools | Tell it not to search in the prompt |
| Every agent has all 30 tools | Role-scoped tool sets of 4–5 | Keep all tools "for flexibility" |
| A generic tool keeps getting bad input | Use the constrained tool that validates its own input | Keep the generic tool and add prompt rules |
| Lookup returns several matching customers | Ask for a distinguishing detail or escalate | Pick the most likely match |
| A pipeline step must always call one tool | Forced `tool_choice` | `auto` and hope |
| Prompt says "analyze content", tool is `analyze_content` | Reword the prompt so it doesn't echo the tool name | Rename the tool at random |

## Exam scenarios that use this domain

- **Scenario 1, customer support agent (D1, D2, D5).** Four MCP tools: `get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`. Trap: a tool that times out returns empty results.
- **Scenario 3, multi-agent research (D1, D2, D5).** Trap: the synthesis subagent queries the web itself instead of going through the coordinator.
- **Scenario 4, developer productivity (D1, D2, D3).** Scope tools by phase (read-only, then add Write and Bash, then deploy tools) and handle overlapping tool descriptions across MCP servers. Trap: giving the agent every tool.

The exam picks 4 of the 6 scenarios. Scenarios 1, 3 and 4 cover Domains 1 and 2 (45% of the exam).

## Latest SDK notes (official docs vs study guide)

The exam likely follows the study guide's model. These are the places where the current official docs differ.

**Corrections**

| Study guide says | Official docs now |
| --- | --- |
| MCP errors have `errorCategory`, `isRetryable`, `retryAfterMs` | The MCP spec defines only **`isError: true`** on a tool result, with `content` and optional `structuredContent`. The category and retry fields are a good **convention** you put inside the content. Unknown tools and invalid arguments are JSON-RPC **protocol errors** (e.g. code `-32602`), not `isError` results. |
| Split generic tools | The official guide also says to **consolidate** related actions on the same resource into one tool with an `action` parameter (e.g. one `pr` tool with create, review, merge). Rule of thumb: split by data source or scope; consolidate actions on one resource. |
| `tool_choice`: auto, any, forced | There is also **`none`**. `any` and `tool` prefill the response, so Claude writes no text before the call, and they skip thinking. They fail with manual extended thinking. On Opus 5.5, Sonnet 5.5, Fable 5.1 and Mythos 5.1, `any` and `tool` return a 400 error: use `auto` with `strict: true`, or structured outputs. |
| `strict: true` enforces priority 1–5 | Strict mode guarantees types, required fields and enums. It does **not** enforce `minimum`/`maximum` or `minLength`/`maxLength`. Use `enum: [1, 2, 3, 4, 5]` or check in code. Every object needs `additionalProperties: false`. At most 20 strict tools per request. |
| 4–5 tools per agent | Still good practice. For very large tool sets, the official answer is the **tool search tool**, which loads tool definitions on demand. |

**Writing tools (official guidance)**

- Description quality is "by far the most important factor in tool performance". Aim for **at least 3–4 sentences**, and include what the tool does **not** return.
- Use `input_examples` for complex or format-sensitive inputs. Each example must validate against the schema.
- Namespace tools that span services: `github_list_prs`, `slack_send_message`. In Claude Code and the Agent SDK, MCP tools are named `mcp__<server>__<tool>`.
- Return high-signal results: stable IDs and only the fields Claude needs for its next step.
- On the Claude API, report a failed tool with `is_error: true` on the `tool_result` block.

**MCP extras worth recognising**

- Tool **annotations**: `readOnlyHint`, `destructiveHint`, `idempotentHint`, `openWorldHint`. They are hints, and clients treat them as untrusted unless the server is trusted. In the Agent SDK, a custom tool marked `readOnlyHint` can run in parallel.
- **`outputSchema`** plus **`structuredContent`** give typed tool results.
- The spec says a human should be able to deny tool calls, and servers must validate inputs and rate-limit calls.

## Sources

- [CCA Prep — Domain 2 overview](https://claudecertprep.com/study-guide/domain2/overview)
- [CCA Prep — Tool descriptions](https://claudecertprep.com/study-guide/domain2/tool-descriptions)
- [CCA Prep — MCP errors](https://claudecertprep.com/study-guide/domain2/mcp-errors)
- [CCA Prep — Tool scoping](https://claudecertprep.com/study-guide/domain2/tool-scoping)
- [CCA Prep — Anti-patterns](https://claudecertprep.com/study-guide/anti-patterns)
- [CCA Prep — Exam scenarios](https://claudecertprep.com/study-guide/exam-scenarios)
- [Claude API — Tool use overview (official)](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview)
- [Claude API — Define tools (official)](https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools)
- [Claude API — Structured outputs (official)](https://platform.claude.com/docs/en/build-with-claude/structured-outputs)
- [MCP specification — Tools](https://modelcontextprotocol.io/specification/2025-06-18/server/tools)

CCA Prep is a community resource, not affiliated with Anthropic.
