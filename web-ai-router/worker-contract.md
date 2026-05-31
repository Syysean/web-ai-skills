# Web AI Worker Contract

Shared output schema for all web AI workers.
All workers must conform to this contract.

---

## Input Schema

```json
{
  "prompt": "string",
  "task_type": "qa | summary | translation | rewrite | extraction | comparison | brainstorming | analysis"
}
```

---

## Output Schema

```json
{
  "site": "gemini | grok | deepseek | claude | chatgpt",
  "task_type": "string",
  "status": "success | partial | error",
  "answer": "string",
  "retry_count": 0,
  "elapsed_ms": 0,
  "error_reason": null
}
```

### Status values

| Value | Meaning |
|-------|---------|
| `success` | Full response extracted and validated |
| `partial` | Response extracted but validation uncertain |
| `error` | Failed after max retries, no usable answer |

### error_reason

Populated only when `status` is `error`:

- `"navigation_failed"`
- `"login_required"`
- `"input_not_found"`
- `"send_failed"`
- `"response_timeout"`
- `"extraction_failed"`

---

## Worker Capabilities

| Worker | Login Required | Strengths | Notes |
|--------|---------------|-----------|-------|
| gemini-worker | ❌ No | General QA, summarization, translation | Guest mode confirmed working |
| grok-worker | ✅ Yes | Real-time web search, current events | Login via Chrome profile |
| deepseek-worker | ✅ Yes | Deep reasoning, math, code analysis | Login via Chrome profile |
| claude-worker | ✅ Yes | Writing, analysis, coding, long-form tasks | Login via Chrome profile |
| chatgpt-worker | ✅ Yes | General QA, writing, coding, creative tasks | Login via Chrome profile |

---

## Compatibility

Any new worker added in the future must output the same JSON schema above.