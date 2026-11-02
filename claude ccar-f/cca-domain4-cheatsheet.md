# CCA Domain 4 Cheat Sheet — Prompt Engineering & Structured Output

Nov 2, 2026 · @Si

## At a glance

Domain 4 is **20%** of the exam. It tests reliable structured output, retry loops that actually fix errors, few-shot examples, choosing between the Batch and synchronous APIs, and splitting long reviews into passes.

| # | Task statement | Section below |
| --- | --- | --- |
| 4.1 | Prompts for reliable structured output with JSON schemas | Structured output |
| 4.2 | Validation retry loops with specific error feedback | Retry loop |
| 4.3 | Few-shot examples for format and style | Few-shot |
| 4.4 | Message Batches API vs synchronous API | Batch API |
| 4.5 | Multi-pass review for complex documents | Multi-pass |

**The five rules that answer most questions**

1. For guaranteed schema compliance, use **`strict: true`** or structured outputs, not stronger wording in the prompt.
2. When validation fails, retry with **specific** feedback: the field, the problem, the bad value and the expected format. Cap the retries, then flag `requires_human_review`.
3. Few-shot examples teach **format, style and how to handle ambiguity**. They never enforce "always" or "never" rules.
4. Use **Batch** only when no one is waiting and the deadline is hours away. It costs about 50% less, takes up to 24 hours, and has no SLA.
5. For a long, complex document, run **separate passes** (extract → verify → score), not one pass that does everything.

## Structured output (4.1)

**Schema in the system prompt (prompt-based)**

- Respond **only** with the JSON object: no preamble, explanation or code fences.
- Use **`null`** for a field you can't determine. Never leave a field out.
- Dates in ISO 8601 (`YYYY-MM-DD`), currency codes in ISO 4217, amounts as plain numbers without symbols.

**`strict: true` on a tool (enforced)**

```python
tools = [{
    "name": "extract_invoice",
    "strict": True,
    "input_schema": {
        "type": "object",
        "properties": {
            "invoice_number": {"type": "string"},
            "currency": {"type": "string", "enum": ["USD", "EUR", "VND"]},
            "total": {"type": "number"},
            "due_date": {"type": ["string", "null"], "format": "date"}
        },
        "required": ["invoice_number", "currency", "total", "due_date"],
        "additionalProperties": False
    }
}]
```

A prompt asks for the right format. `strict: true` makes the API enforce it.

## Retry loop (4.2)

```python
for attempt in range(3):
    raw = call_claude(messages)
    try:
        data = json.loads(raw)
        validate(data, SCHEMA)        # jsonschema, plus business rules
        return {"status": "ok", "data": data}
    except (json.JSONDecodeError, ValidationError) as e:
        messages += [
            {"role": "assistant", "content": raw},
            {"role": "user", "content":
             f"Field 'invoice_date' is invalid: got '03/15/24'. "
             f"Expected ISO 8601 YYYY-MM-DD, e.g. '2024-03-15'. Fix only that field."}]
return {"status": "failed", "requires_human_review": True, "error": str(e), "partial": raw}
```

- Good feedback names the **field**, the **problem**, the **wrong value** and the **expected format with an example**.
- "That was invalid JSON, try again" gives the model nothing to fix.
- **Graceful degradation:** in a batch, documents that keep failing go to a `needs_review` list with the error and partial data. They are never dropped.

## Few-shot (4.3)

| Few-shot works for | Few-shot does not guarantee |
| --- | --- |
| Output format and schema | Rule compliance |
| Reasoning patterns | Tool ordering |
| Ambiguous edge cases | Policy enforcement |
| Tone and style | Preventing prohibited behavior |
| Classifying borderline inputs | Deterministic workflow control |

**Exam rule:** if a requirement says "always", "never" or "every time", the answer is programmatic enforcement (a hook, `strict`, or code), not more examples.

**Ambiguity example.** "Charged twice and now my account is locked" → `billing` (primary), with account access as secondary context. Each example gets a one-line reason before the output.

**CI review calibration.** Show one **good** comment (names the line, the vulnerability, the cause and a concrete fix) and one **bad** comment (vague, "looks risky"). Add a **known-safe list**, such as `eval()` in the sandbox directory or dynamic `require()` in build tooling.

## Batch API (4.4)

| | Batches API | Synchronous API |
| --- | --- | --- |
| Cost | About 50% less | Full price |
| Latency | Up to 24 hours | Seconds |
| SLA | None | Standard |
| User waiting? | No | Yes |
| Volume | High (100+ requests) | Any |

**Use Batch only if all are true:** no one is waiting, the deadline is hours away, the volume is high, and the savings are worth the unpredictable finish time.

**Never use Batch for:** live chat or search, a pipeline step that blocks other work, a hard deadline ("results by 8 AM"), or small one-off jobs.

```python
batch = client.messages.batches.create(requests=[
    {"custom_id": t.id, "params": {"model": "claude-haiku-5-5", "max_tokens": 100,
        "system": CLASSIFY_PROMPT, "messages": [{"role": "user", "content": t.text}]}}
    for t in tickets])
# later: when processing_status == "ended"
for r in client.messages.batches.results(batch.id):
    if r.result.type == "succeeded": save(r.custom_id, r.result.message)
    else: log_failure(r.custom_id, r.result.type)
```

Example from the guide: classifying 50,000 support tickets overnight costs about $20 in a batch instead of about $40 synchronously.

## Multi-pass review (4.5)

For an 80-page document, give each pass one job:

1. **Extract claims**: no judgment.
2. **Verify** each claim against its sources.
3. **Score** credibility and synthesize.
4. **Report** with citations.

One pass that does all three dilutes attention, and all three come out worse.

## Anti-patterns and exam triggers

| If the question says… | The answer is… | Trap answer |
| --- | --- | --- |
| Output sometimes breaks the schema | `strict: true` / structured outputs | "Respond ONLY with valid JSON!!" |
| Retries don't fix the error | Feedback naming field, problem, bad value, expected format | "Invalid, try again" |
| A document fails validation 3 times | `requires_human_review` + partial data to a review queue | Drop it, or retry forever |
| Tools must run in a fixed order | Programmatic prerequisite (hook) | More few-shot examples of the order |
| Model invents values to fill fields | Make unknowable fields nullable (`null` allowed) | Make every field required |
| Ambiguous tickets classified inconsistently | Few-shot examples with a short reason each | A longer rule list |
| "Save money, send everything to Batch" | Batch only for non-urgent background work | Batch for chat or blocking steps |
| Results needed by a fixed morning deadline | Synchronous (or Batch started very early, with slack) | Batch submitted late, with no SLA |
| Huge batch job is about to be submitted | Test the prompt on a small sample first | Submit all 50,000 straight away |
| One prompt extracts, verifies and scores | Separate passes | A longer single prompt |

## Exam scenarios that use this domain

- **Scenario 5, Claude Code for CI/CD (D3, D4).** Few-shot good/bad review comments and a known-safe list. Trap: generic prompts that give generic reviews.
- **Scenario 6, structured data extraction (D4, D5).** Retry loops with specific errors, `requires_human_review` for persistent failures, per-section extraction for long documents, Batch for bulk work. Traps: generic retry feedback; Batch for time-sensitive systems; one call for a whole long document.

## Latest SDK notes (official docs vs study guide)

The exam likely follows the study guide's model. These are the places where the current official docs differ.

**Corrections**

| Study guide says | Official docs now |
| --- | --- |
| `strict: true` on a tool is the way to enforce a schema | There are two GA features, available on all models: **JSON outputs**, via `output_config.format = {"type": "json_schema", "schema": ...}` (SDK helper: `client.messages.parse(...)`), and **strict tool use** (`strict: true`). The old `output_format` parameter is deprecated. |
| `strict` guarantees schema compliance | It guarantees valid JSON, types, required fields and enums. It does **not** enforce `minimum`/`maximum`, `minLength`/`maxLength` or complex regex. Every object needs `additionalProperties: false`. A refusal or `max_tokens` stop can still break the schema, and enum letter case isn't guaranteed. Limits: 20 strict tools and 24 optional parameters per request. It can't be combined with citations or prefill. |
| The retry loop fixes invalid JSON | With structured outputs, JSON syntax errors basically disappear. The retry loop now matters for **semantic checks**: totals that add up, dates in range, cross-field rules. |
| "Respond ONLY with JSON" (and prefilling `{`) | **Prefilling the last assistant turn returns a 400 error on Claude 4.6 and newer models.** Use structured outputs instead. |
| `client.beta.messages.batches.create` | The standard path is **`client.messages.batches.create`**; the beta path is only for special headers. A batch holds up to **100,000 requests or 256 MB**. Most finish **in under 1 hour**; a batch **expires at 24 hours**. Results are kept **29 days**. |
| Match results by `custom_id` | Results are **not in request order**, so this is required. `custom_id` must be unique, 1–64 characters, matching `[a-zA-Z0-9_-]`. Result types are `succeeded`, `errored`, `canceled` and `expired`; only `succeeded` is billed. |
| Test before a big batch (anti-pattern list) | Official docs agree: validation errors appear **only after the whole batch ends**, so dry-run one request with the Messages API first. Batches don't support streaming. Prompt caching works in batches on a best-effort basis (a 1-hour TTL is recommended), and structured outputs work too. |
| Haiku at $0.80 / $4 per million tokens | Prices change; check current pricing. The 50% batch discount is the stable fact. |
| Few-shot (no count given) | Official guidance: **3–5 examples**, wrapped in `<example>` tags (several go inside `<examples>`). Make them relevant to the real task and diverse enough to cover edge cases. |
| Multi-pass review | The official name is **prompt chaining**. The most common chain is self-correction: draft → review against criteria → refine, with each step a separate call you can log and inspect. |

**Official long-document tips (often tested)**

- Put long documents **at the top** and the question or instructions **at the end**. This can improve answers by up to 30%.
- Wrap each document in `<document>` tags with `<document_content>` and `<source>` subtags.
- Ask Claude to **quote the relevant parts first**, then do the task.
- Be explicit, and give the reason behind a rule. Claude generalizes from the reason.

## Sources

- [CCA Prep — Domain 4 overview](https://claudecertprep.com/study-guide/domain4/overview)
- [CCA Prep — Structured output](https://claudecertprep.com/study-guide/domain4/structured-output)
- [CCA Prep — Batch API](https://claudecertprep.com/study-guide/domain4/batch-api)
- [CCA Prep — Few-shot](https://claudecertprep.com/study-guide/domain4/few-shot)
- [CCA Prep — Anti-patterns](https://claudecertprep.com/study-guide/anti-patterns)
- [CCA Prep — Exam scenarios](https://claudecertprep.com/study-guide/exam-scenarios)
- [Claude API — Structured outputs (official)](https://platform.claude.com/docs/en/build-with-claude/structured-outputs)
- [Claude API — Batch processing (official)](https://platform.claude.com/docs/en/build-with-claude/batch-processing)
- [Claude API — Prompting best practices (official)](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices)

CCA Prep is a community resource, not affiliated with Anthropic.
