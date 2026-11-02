# CCA Domain 1 Cheat Sheet — Agentic Architecture & Orchestration

Nov 2, 2026 · @Si

## At a glance

Domain 1 is the heaviest domain on the exam at **27%**. It tests whether you can design, run and debug agents built on the Claude Agent SDK.

| # | Task statement | Section below |
| --- | --- | --- |
| 1.1 | Design and implement agentic loops | Agentic loop |
| 1.2 | Orchestrate coordinator-subagent systems | Multi-agent |
| 1.3 | Configure subagent invocation, context passing, spawning | Multi-agent |
| 1.4 | Multi-step workflows with enforcement and handoff | Hooks |
| 1.5 | Hooks for tool call interception and data normalization | Hooks |
| 1.6 | Task decomposition strategies | Task decomposition |
| 1.7 | Session state, resumption, forking | Sessions |

**The five rules that answer most questions**

1. End the loop on `stop_reason == "end_turn"`, never on the text Claude wrote.
2. Subagents only talk through the coordinator and see only what it passes them.
3. Spawn parallel subagents with several Task calls in **one** coordinator turn.
4. A requirement with "always / never / must" belongs in a **hook**, not the prompt.
5. Stale context means a fresh session plus a structured summary, not `--resume`.

## Agentic loop (1.1)

The loop keeps calling Claude until `stop_reason` says it is done. Know the order exactly.

1. Send messages + tool definitions to Claude.
2. Check `stop_reason`:
   - `end_turn` → done, return the answer.
   - `tool_use` → continue to step 3.
   - `max_tokens` or anything else → handle it explicitly (raise, continue, or truncate).
3. Append Claude's **assistant** message to history (it contains the `tool_use` blocks).
4. Run **every** `tool_use` block in the response.
5. Append **all** results together as **one user** message of `tool_result` blocks, each matched by `tool_use_id`.
6. Go back to step 1.

```python
while True:
    resp = client.messages.create(model=..., tools=tools, messages=messages, max_tokens=4096)
    if resp.stop_reason == "end_turn":
        return resp.content
    if resp.stop_reason != "tool_use":
        raise RuntimeError(resp.stop_reason)
    messages.append({"role": "assistant", "content": resp.content})   # first
    results = [{"type": "tool_result", "tool_use_id": b.id, "content": run(b.name, b.input)}
               for b in resp.content if b.type == "tool_use"]
    messages.append({"role": "user", "content": results})             # second
```

**Rules to remember**

- Assistant message **before** tool results. Skipping it breaks user/assistant alternation and the API rejects the call.
- One response can hold several `tool_use` blocks. Run them all, then send all results in a single user message.
- `max_turns` and `max_budget_usd` (Agent SDK options) are **safety caps** for runaway loops, never the primary stop condition.
- Never decide to stop by checking text such as `"Task complete" in text`. It is fragile and non-deterministic.
- `stop_reason == "max_tokens"` mid tool call: the tool input may be cut off, so don't run it. Retry with a higher limit or surface an error.
- A tool that throws still gets a `tool_result` (with `is_error: true`). Every `tool_use_id` needs an answer, or the next call is rejected.

## Multi-agent orchestration (1.2, 1.3)

The exam's pattern is **hub-and-spoke**: one coordinator, many isolated subagents.

**Invariants**

- Subagents never talk to each other. Everything goes through the coordinator.
- Each subagent has **isolated context**. It sees only what the coordinator passes in its prompt, never the coordinator's history.
- The coordinator owns decomposition, delegation, aggregation, quality checks and error routing.

**Spawning**

- The coordinator needs `"Task"` in `allowed_tools`. Without it, it cannot spawn subagents. *(See Latest SDK notes: now `Agent`.)*
- **Parallel** = several Task calls in **one** coordinator response. **Sequential** = one Task per turn, which is slow.

**Context passing**

- "Synthesize the findings" with no data is wrong: the subagent has nothing to work with.
- Pass **structured** data (JSON with `content`, `source_url`, `source_title`, `retrieved_at`), not a concatenated text blob. Structure keeps attribution.
- State the requirements in the prompt: cite each claim, cover all findings, flag contradictions.

**Routing and quality**

| Choice | Wrong | Right |
| --- | --- | --- |
| Routing | Fixed: always call every subagent | Dynamic: coordinator plans, calls only what the query needs |
| Quality | Hope the first pass is enough | Iterative refinement: set criteria, evaluate, re-delegate targeted gap-filling queries |

**Error handling**

- The subagent retries transient errors locally first (for example 3 tries with exponential backoff).
- If it still fails, it returns a **structured error** to the coordinator: `status: partial_failure`, results gathered so far, the error, `is_retryable`, and what it attempted.
- Never fail silently and never return only a bare error string. The coordinator needs enough to decide what to do next.

## Hooks and enforcement (1.4, 1.5)

Hooks are your code, run by the **SDK** (not by Claude) at fixed points in the loop. They are deterministic, so Claude can't skip them.

| | Hooks | Prompt instructions |
| --- | --- | --- |
| Reliability | Runs every time | Probabilistic, ~1–3% misses on compliance paths |
| Use for | Business rules, policy, data normalization | Style, tone, guidance |
| Auditability | Logged as code | Invisible reasoning |
| Long context | Can't be forgotten | Can be "forgotten" |

**Exam trigger:** "always", "never", "must", "every time" in a requirement → use a hook.

**The two hooks the exam focuses on**

| Hook | Fires | Use for |
| --- | --- | --- |
| `PreToolUse` (the "interception" hook) | Before the tool runs | Block policy violations (refund > $500 → escalate), path limits, rate limits, prerequisite gates |
| `PostToolUse` | After the tool runs, before Claude sees the result | Normalize data (Unix time → ISO 8601, status codes → words, currency), audit logging |

The full SDK has 33 hook events (10 in Python). You only need these two in depth for Domain 1.

**Prerequisite gate (canonical exam pattern)**

- `PostToolUse` on `get_customer` records `verified = True` in session state.
- `PreToolUse` on `process_refund` / `process_payment` / `update_account` denies the call until `verified` is true.
- The deny carries a clear reason ("call get_customer first") so Claude can recover by itself.

**Real SDK syntax** (the study guide's `pre_tool_use_hooks=` parameter does not exist)

```python
from claude_agent_sdk import ClaudeAgentOptions, HookMatcher

state = {"verified": False}

async def gate(input_data, tool_use_id, context):
    if not state["verified"]:
        return {"hookSpecificOutput": {
            "hookEventName": "PreToolUse",
            "permissionDecision": "deny",
            "permissionDecisionReason": "Verify identity: call get_customer first"}}
    return {}  # allow

async def track(input_data, tool_use_id, context):
    state["verified"] = True  # check the tool result in real code
    return {}

options = ClaudeAgentOptions(hooks={
    "PreToolUse":  [HookMatcher(matcher="process_refund|process_payment|update_account", hooks=[gate])],
    "PostToolUse": [HookMatcher(matcher="get_customer", hooks=[track])],
})
```

- To block, return `permissionDecision: "deny"` with a reason. To allow, return `{}`.
- When several `PreToolUse` hooks disagree: **deny > defer > ask > allow**.
- A matcher filters on the tool name only. A `Write|Edit` matcher does not catch a Bash command that writes the same file.
- MCP tools are named `mcp__<server>__<tool>`, so match that name.

## Session management (1.7)

Pick the operation by one question: is the prior context still valid?

| Operation | Use when |
| --- | --- |
| `--resume <session-name>` | Prior context is still valid and you're continuing the same work |
| `fork_session` | You want two independent approaches from the same baseline (e.g. Repository vs Service Locator) |
| New session + injected summary | Prior tool results are stale or a core assumption changed |

**Signs the context is stale → start fresh**

- A fundamental assumption turned out wrong.
- A major architecture decision changed.
- 50%+ of the files Claude analyzed have been modified.
- The new task is in a different domain from the old session.

**After `--resume`, always say what changed.** Claude does not detect file changes on its own. List the changed files and ask it to re-analyze them, or it reasons from stale analysis.

**Structured summary for a fresh session** carries only what's worth keeping:

- What was learned (facts discovered, e.g. JWT is RS256, not HS256).
- Decisions made (e.g. Adapter pattern for backward compatibility).
- The starting point and the do-not-touch list.

**Multi-day pattern:** explore (named session) → resume and fork into design-a / design-b → pick one → implement in a **new** session seeded with the design summary, so dead ends don't clutter the context.

## Task decomposition (1.6)

Choose by whether you know the steps upfront and whether they depend on each other.

| Pattern | Use when | Example |
| --- | --- | --- |
| Prompt chaining | Steps are known upfront and run in order | File-by-file analysis → cross-file synthesis |
| Dynamic adaptive | Scope is unknown until you explore | "Add tests to a legacy codebase" |
| Parallel decomposition | Subtasks are independent | Web search + document analysis at the same time |

## Anti-patterns and exam triggers

Most wrong answer choices are one of these anti-patterns.

| If the question says… | The answer is… | Trap answer |
| --- | --- | --- |
| How should the loop know it's done? | `stop_reason == "end_turn"` | Check the text for "done", or rely on `max_turns` |
| Claude returned several tool calls | Run all, return all results in one user message | Run only the first, or one message per result |
| Subagents need each other's results | Route through the coordinator | Let subagents call each other |
| The subagent's output is vague or wrong | Pass explicit, structured context | Assume it can see the coordinator's history |
| Two independent subtasks are slow | Several Task calls in one coordinator turn | One Task per turn |
| Not every query needs every subagent | Dynamic routing | Fixed pipeline that always calls all |
| A rule must **always / never** apply | Hook (`PreToolUse` to block, `PostToolUse` to normalize) | Stronger wording in the system prompt |
| Step B must only run after step A | Prerequisite gate hook | "Please verify first" in the prompt |
| Tool data formats are inconsistent | `PostToolUse` normalization | Ask Claude to convert formats |
| Subagent hit a timeout | Retry locally, then structured partial-failure error | Fail silently or crash the whole run |
| Resuming after files changed | `--resume` + list the changed files | Resume silently |
| Core assumption was wrong | New session + structured summary | `--resume` the old session |
| Compare two approaches | `fork_session` from the same baseline | Run both in one session |
| Tools must run in a fixed order | Prerequisite hook | Few-shot examples of the order |
| One subagent failed | Structured partial failure; coordinator decides | Return empty results, or end the whole workflow |

**The meta-pattern behind most anti-patterns:** a probabilistic mechanism is used where a deterministic one is needed.

| Task | Probabilistic (wrong) | Deterministic (right) |
| --- | --- | --- |
| Tool ordering | Few-shot examples | Prerequisite hooks |
| Escalation | Self-reported confidence | Rule-based triggers |
| Schema compliance | Prompt instructions | `strict: true` + JSON schema |
| Loop termination | Parsing the text | `stop_reason` |
| Error recovery | Retry everything | `isRetryable` flag |

## Exam scenarios that use this domain

- **Scenario 1, customer support agent (D1, D2, D5).** Gate `process_refund` behind a verified `get_customer` with a prerequisite hook. Split a customer with several issues into parallel tracks. Trap: enforcing the order through the system prompt.
- **Scenario 3, multi-agent research (D1, D2, D5).** Run web search and document analysis in parallel. Pass findings with source metadata. The coordinator evaluates the synthesis and re-delegates to fill gaps. Traps: routing every query through all four subagents; plain-text findings.
- **Scenario 4, developer productivity (D1, D2, D3).** Use dynamic, adaptive decomposition for open-ended tasks like "add tests to this legacy codebase". Trap: a fixed sequence of steps.

The exam picks 4 of the 6 scenarios. Scenarios 1, 3 and 4 cover Domains 1 and 2 (45% of the exam), so study them hardest.

## Latest SDK notes (official docs vs study guide)

The exam likely follows the study guide's model. These are the places where the current official Agent SDK docs differ, so you recognise both names.

**Corrections**

| Study guide says | Official SDK now |
| --- | --- |
| Spawn subagents with the `Task` tool | The tool is now **`Agent`**. `tool_use` blocks show `"Agent"`; the init tools list still shows `"Task"`. Accept both. |
| `"Task"` must be in `allowed_tools` to spawn | `allowed_tools` only **auto-approves** a tool. Unlisted tools stay available and go through permission checks. |
| `fork_session` is a standalone call | An option used with resume: `resume=session_id, fork_session=True`. The fork gets a new session ID; the original is untouched. |
| `claude --session-name ...` | Not confirmed in the SDK docs. The SDK uses `resume=<id>`, or `continue_conversation=True` (Python) / `continue: true` (TS) for the most recent session. |
| Subagents never talk to each other | True for the exam's hub-and-spoke. In the SDK, subagents can spawn their own subagents (default depth 3), and one with `SendMessage` can message other named agents. |

**Defining subagents**

- Pass `agents={...}` with an `AgentDefinition` per subagent: `description` (required), `prompt` (required), and optional `tools`, `model`, `maxTurns`, `effort`, `permissionMode`.
- Claude picks a subagent from its `description`. To force one, name it in the prompt ("Use the code-reviewer agent...").
- A subagent receives only its own `prompt`, the Agent tool's prompt string, project CLAUDE.md, and its tools. It does **not** get the parent's history or system prompt.
- Only the subagent's final message returns to the parent, which keeps the parent's context small.
- Subagents run in the background by default. Claude waits when its next step needs the report.

**The loop in the SDK**

- The SDK runs the loop for you. You read the final `ResultMessage.subtype` instead of checking `stop_reason` yourself.
- Subtypes: `success`, `error_max_turns`, `error_max_budget_usd`, `error_during_execution`, `error_max_structured_output_retries`. Only `success` has a `result` field.
- `stop_reason` on the result can be `end_turn`, `max_tokens` or `refusal`.
- `max_turns` counts tool-use turns only. `max_budget_usd` includes subagent spend; at the cap, new subagents are refused.
- In one turn, read-only tools (`Read`, `Glob`, `Grep`) run in parallel; `Edit`, `Write`, `Bash` run one at a time.

**Sessions and context**

- Sessions persist the **conversation**, not the filesystem. A fork does not branch files; file checkpointing does. That is why you must tell Claude what changed after a resume.
- Capture the ID from `ResultMessage.session_id` to resume later.
- When context fills up, the SDK auto-compacts older history into a summary. Put persistent rules in CLAUDE.md, which is re-injected every request. A `PreCompact` hook can archive the transcript first.

## Sources

- [CCA Prep — Domain 1 overview](https://claudecertprep.com/study-guide/domain1/overview)
- [CCA Prep — Agentic loops](https://claudecertprep.com/study-guide/domain1/agentic-loops)
- [CCA Prep — Multi-agent systems](https://claudecertprep.com/study-guide/domain1/multi-agent)
- [CCA Prep — Hooks and enforcement](https://claudecertprep.com/study-guide/domain1/hooks)
- [CCA Prep — Session management](https://claudecertprep.com/study-guide/domain1/session-management)
- [CCA Prep — Anti-patterns](https://claudecertprep.com/study-guide/anti-patterns)
- [CCA Prep — Exam scenarios](https://claudecertprep.com/study-guide/exam-scenarios)
- [Claude Agent SDK — Hooks (official)](https://code.claude.com/docs/en/agent-sdk/hooks)
- [Claude Agent SDK — How the agent loop works (official)](https://code.claude.com/docs/en/agent-sdk/agent-loop)
- [Claude Agent SDK — Subagents (official)](https://code.claude.com/docs/en/agent-sdk/subagents)
- [Claude Agent SDK — Sessions (official)](https://code.claude.com/docs/en/agent-sdk/sessions)

CCA Prep is a community resource, not affiliated with Anthropic. Hook syntax here follows the official SDK docs.
