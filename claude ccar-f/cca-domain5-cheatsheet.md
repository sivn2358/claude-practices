# CCA Domain 5 Cheat Sheet — Context Management & Reliability

Nov 2, 2026 · @Si

## At a glance

Domain 5 is **15%** of the exam. It tests what happens when things get long or go wrong: long documents, escalating to humans, keeping track of sources, surviving failures, and caching for cost.

| # | Task statement | Section below |
| --- | --- | --- |
| 5.1 | Context window strategies for long documents | Long context |
| 5.2 | Reliable escalation without self-reported confidence | Escalation |
| 5.3 | Preserve provenance across multi-agent handoffs | Provenance |
| 5.4 | Graceful degradation and error resilience | Resilience |
| 5.5 | Cost optimization with prompt caching | Prompt caching |

**The five rules that answer most questions**

1. Details lost in the middle of a long document → **per-section passes plus an integration pass**. A bigger context window is the trap answer.
2. Escalate on **programmatic rules** (thresholds, missing data, error counts, a category list), never on the model's stated confidence or on sentiment.
3. A handoff to a human must be **self-contained**: identity, issue, root cause, actions taken, escalation reason, recommended next step.
4. Every claim carries its **source** (ID, URL, title, date, page). The synthesis cites source IDs and **says plainly which topics have no sources**.
5. Cache only a **byte-identical prefix**: no timestamps or versions inside it. Isolate failures per item, and use a circuit breaker for a service that keeps failing.

## Long context (5.1)

**Attention dilution ("lost in the middle").** Models attend reliably to the start and end of a long context and less reliably to the middle. This comes from how attention works, not from the window size, so a larger window only moves the weak zone.

**Per-section pass architecture**

1. **Pass 1:** one focused call per section extracts claims, dates, entities and key facts. Each result keeps the section index and a short reference snippet.
2. **Pass 2:** one integration call reads all the section results and finds cross-section themes, contradictions and the overall story.

**Rolling summary for long conversations**

- Below a threshold (e.g. 40 messages), keep the history unchanged.
- Above it, summarize the older messages and keep the last ~20 turns word for word.
- New history = summary message + short acknowledgment + recent turns.

## Escalation (5.2)

Models are poorly calibrated: they can sound sure on hard questions and hedge on easy ones. The cases that most need a human are the ones the model is most likely to say it can handle.

```python
def should_escalate(s):
    if s.refund_amount > 500:            return True, "refund_over_limit"
    if not s.customer_verified:          return True, "identity_not_verified"
    if s.tool_error_count > 3:           return True, "excessive_tool_failures"
    if s.category in {"fraud", "legal", "executive_complaint"}:
                                         return True, "category_requires_human"
    return False, None
```

**Handoff record (the human has no access to the session)**

| Group | Fields |
| --- | --- |
| Identity | customer ID, name, account tier |
| Issue | summary, root cause found |
| Actions taken | what was done, credits applied |
| Escalation | reason, rule that fired, recommended next action |
| Metadata | session start, escalation time, session reference |

The human should never need to read the conversation or ask the customer again.

## Provenance (5.3)

```json
{"claim": "Q3 revenue rose 12%",
 "source_id": "S4", "url": "https://...", "title": "Q3 report",
 "retrieved_at": "2026-11-02", "page": 7}
```

- Subagents pass findings as structured records like this, not plain text.
- Each section of the synthesis lists the `citation_ids` it relies on.
- If a topic has no source coverage, the report **says so**. It doesn't silently leave the topic out.

## Resilience (5.4)

**Per-item error isolation.** Wrap each document in its own `try`. A failure marks that item `validation_failed` or `error` with `requires_human_review: true`, and the batch carries on. Report total, succeeded and failed counts.

**Circuit breaker**

| State | Behavior | Transition |
| --- | --- | --- |
| Closed | Calls go through | Opens after 5 failures |
| Open | Calls fail fast ("service unavailable") and ops is alerted | Goes half-open after 60 s |
| Half-open | One trial call is allowed | Success → closed; failure → open again |

**Subagent failure.** Return a structured partial failure to the coordinator. Don't end the whole workflow, and don't return empty results.

## Prompt caching (5.5)

```python
system=[{"type": "text", "text": BIG_STABLE_PROMPT,
         "cache_control": {"type": "ephemeral"}}]
```

- Cache a **large, stable prefix**: system prompt, few-shot examples, reference documents.
- It pays off when many requests share that prefix.
- **Any change**, even one character, invalidates the cache, including a timestamp, version string or extra space. Keep the prefix immutable and put changing content after it.

## Anti-patterns and exam triggers

| If the question says… | The answer is… | Trap answer |
| --- | --- | --- |
| Agent misses details in the middle of a 200-page doc | Per-section passes + integration pass | A model with a bigger context window |
| When should the agent escalate? | Rule-based triggers in code | When Claude says it's under 70% confident |
| Angry customers should go to humans | Rules on category, thresholds and missing data | Sentiment analysis as the router |
| The human agent keeps re-asking the customer | Self-contained structured handoff | Send a link to the transcript |
| Report has claims with no sources | Structured findings with source metadata, citation IDs | Plain-text summaries from subagents |
| A topic has no sources | State the gap in the report | Leave the topic out silently |
| 1 of 500 documents throws an error | Isolate it, flag for review, continue | Stop the whole batch |
| A downstream service keeps timing out | Circuit breaker + alert | Retry forever |
| 97% overall accuracy, so cut human review? | Check accuracy per document type and field first | Trust the aggregate number |
| Lookup returns 3 customers with the same name | Ask for a distinguishing detail or escalate | Pick the most likely one |
| Cache hit rate is near zero | Remove the date/version from the cached prefix | Add more cache breakpoints |
| Chat history keeps growing | Rolling summary + recent turns verbatim | Send the full history forever |

**The meta-pattern behind most anti-patterns:** a probabilistic mechanism is used where a deterministic one is needed.

| Task | Probabilistic (wrong) | Deterministic (right) |
| --- | --- | --- |
| Tool ordering | Few-shot examples | Prerequisite hooks |
| Escalation | Self-reported confidence | Rule-based triggers |
| Schema compliance | Prompt instructions | `strict: true` + JSON schema |
| Loop termination | Parsing the text | `stop_reason` |
| Error recovery | Retry everything | `isRetryable` flag |

## Exam scenarios that use this domain

- **Scenario 1, customer support agent (D1, D2, D5).** Rule-based escalation and self-contained handoffs. Trap: escalating because Claude sounds unsure.
- **Scenario 2, code generation with Claude Code (D3, D5).** Resume vs start fresh in multi-day projects. Trap: continuing after a core assumption changed.
- **Scenario 3, multi-agent research (D1, D2, D5).** Provenance metadata and partial subagent failures. Trap: plain-text findings that lose attribution.
- **Scenario 6, structured data extraction (D4, D5).** Per-section passes for long documents and `requires_human_review` for failures. Trap: one call for a whole long document.

## Latest SDK notes (official docs vs study guide)

The exam likely follows the study guide's model. These are the places where the current official docs differ or add detail.

**Corrections and additions**

| Study guide says | Official docs now |
| --- | --- |
| A bigger window doesn't fix dilution | The official docs agree and call it **context rot**: accuracy and recall degrade as token count grows, so "more context isn't automatically better". Current models have a **1M-token** window by default, which makes this trap more tempting, not less. |
| (not covered) prompt layout | Official long-context tips: put long documents **at the top** and the question **at the end** (up to 30% better), wrap documents in `<document>` tags with `<source>`, and ask for **relevant quotes first**. |
| Hand-written rolling summaries | The platform has **server-side compaction** (beta, Claude 4.6+), which summarizes earlier turns automatically, and **context editing** (clearing old tool results or thinking blocks). The Agent SDK and Claude Code auto-compact too. Put rules that must survive in CLAUDE.md, which is re-injected every request. |
| (not covered) overflow | Input larger than the window → 400 "prompt is too long". Input plus `max_tokens` over the window → `stop_reason: "model_context_window_exceeded"` (Claude 4.5+). The API never silently truncates. |
| `cache_control` on the system prompt | There is also **automatic caching**: a top-level `cache_control` that moves the breakpoint forward as the conversation grows. Up to **4 breakpoints** per request. Cache order is **tools → system → messages**. Changing tool definitions invalidates everything; changing `tool_choice` or images invalidates the messages part. |
| Caching cuts cost | TTL is **5 minutes** by default (cache write costs 1.25× normal input) or **1 hour** with `"ttl": "1h"` (2×). Cache reads cost **0.1×** normal input (0.05× on Opus 5.5 and Sonnet 5.5). The minimum cacheable prefix is **512 tokens** on current models; a shorter prefix is silently not cached. Caching saves cost and latency but **not context space**. Check `cache_read_input_tokens` to confirm hits. |
| Any change breaks the cache | Correct: the prefix must be 100% identical up to the breakpoint. Put the breakpoint on the last block that stays the same between requests. A breakpoint on a block that changes every time writes a new entry and never gets a read. |

Escalation, provenance, per-item isolation and circuit breakers are architecture patterns. The official docs don't contradict the study guide on them.

## Sources

- [CCA Prep — Domain 5 overview](https://claudecertprep.com/study-guide/domain5/overview)
- [CCA Prep — Context strategies](https://claudecertprep.com/study-guide/domain5/context-strategies)
- [CCA Prep — Escalation](https://claudecertprep.com/study-guide/domain5/escalation)
- [CCA Prep — Anti-patterns](https://claudecertprep.com/study-guide/anti-patterns)
- [CCA Prep — Exam scenarios](https://claudecertprep.com/study-guide/exam-scenarios)
- [Claude API — Context windows (official)](https://platform.claude.com/docs/en/build-with-claude/context-windows)
- [Claude API — Prompt caching (official)](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- [Claude API — Prompting best practices (official)](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices)

CCA Prep is a community resource, not affiliated with Anthropic.
