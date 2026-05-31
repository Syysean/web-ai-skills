---
name: web-ai-router
description: "Use this skill ONLY when the user explicitly asks to use Gemini, Grok, DeepSeek, Claude, or ChatGPT by name, or says 'ask another AI', 'let Grok search', 'use DeepSeek to analyze', 'ask Claude', 'query an external AI'. Do NOT trigger for general search or QA tasks."
---

# Web AI Router

Select the best available web AI for a task and delegate to the appropriate worker.

---

## Available Workers

| Worker | URL | Login Required | Best For |
|--------|-----|---------------|----------|
| gemini-worker | gemini.google.com | ❌ No | General QA, summarization, translation, current events (has Google Search) |
| grok-worker | grok.com | ✅ Yes | Real-time web search, breaking news, current events, X/Twitter context |
| deepseek-worker | chat.deepseek.com | ✅ Yes | Deep reasoning, math, code analysis, complex multi-step problems |
| claude-worker | claude.ai | ✅ Yes | Writing, analysis, coding, nuanced long-form tasks |
| chatgpt-worker | chatgpt.com | ✅ Yes | General QA, writing, coding, creative tasks |

See `worker-contract.md` for the shared output schema.

---

## Routing Logic

### Step 1 — Identify task type

Read the user's request and classify:

| Task | Best Worker | Reason |
|------|-------------|--------|
| Current news, real-time events | grok-worker | Live web search |
| Math, code, logic, deep analysis | deepseek-worker | Strong reasoning |
| Writing, editing, long-form tasks | claude-worker | Nuanced language |
| Creative tasks, general coding | chatgpt-worker | Broad capability |
| General QA, translation, summary | gemini-worker | Fast, no login needed |
| Ambiguous or mixed | gemini-worker | Default fallback |

### Step 2 — Check worker availability

Before routing, verify the target worker is accessible:

- gemini-worker: always available (no login required)
- grok-worker: requires Grok session in Chrome profile
- deepseek-worker: requires DeepSeek session in Chrome profile
- claude-worker: requires Claude session in Chrome profile
- chatgpt-worker: requires ChatGPT session in Chrome profile

If the preferred worker is unavailable (login required but session lost), fall back to gemini-worker and note the fallback in the output.

### Step 3 — Route to worker

Load and execute the appropriate worker skill:

```
[skill] gemini-worker   → for general tasks
[skill] grok-worker     → for real-time search tasks
[skill] deepseek-worker → for reasoning/analysis tasks
[skill] claude-worker   → for writing/long-form tasks
[skill] chatgpt-worker  → for creative/general tasks
```

### Step 4 — Return unified output

All workers return the same schema (defined in worker-contract.md):

```json
{
  "site": "gemini | grok | deepseek | claude | chatgpt",
  "task_type": "qa | summary | translation | rewrite | extraction | comparison | brainstorming | analysis",
  "status": "success | partial | error",
  "answer": "<response text>",
  "retry_count": 0,
  "elapsed_ms": 0,
  "error_reason": null
}
```

Follow with a short human-readable summary.

---

## Multi-Worker Mode

If the user asks to query multiple AIs and compare results, run workers sequentially and present results side by side.

Example trigger: "ask both Gemini and Grok about X and compare"

Output format for multi-worker:

```
## Gemini
<answer>

## Grok
<answer>

## Comparison
<brief synthesis>
```

---

## Decision Examples

| User request | Route to |
|-------------|----------|
| "What are the latest AI news?" | grok-worker (real-time search) |
| "Can you summarize this article?" | gemini-worker (fast summarization) |
| "Help me debug this code" | deepseek-worker (code analysis) |
| "Rewrite this paragraph to be more formal" | claude-worker (nuanced writing) |
| "Ask both Gemini and Grok about the latest news and compare" | gemini-worker + grok-worker (multi-worker mode) |
| "Translate this text to French" | gemini-worker (translation) |
| "Analyze this legal document" | claude-worker (long-form analysis) |
| "What does this code do?" | deepseek-worker (code understanding) |
| "Write a poem about the ocean" | chatgpt-worker (creative writing) |
| "What's the stock price of X?" | grok-worker (real-time search) |
| "Summarize this research paper" | gemini-worker (summarization) |
| "Compare the latest news on X from Gemini and Grok" | gemini-worker + grok-worker (multi-worker comparison) |
| "Ask Claude to rewrite this email" | claude-worker (writing tasks) |