# web-ai-skills

**Pi agent skills for sending prompts to web-based AI chatbots via browser automation — no API keys required.**

[中文文档](./README.zh-CN.md)

---

https://github.com/user-attachments/assets/48414fa1-4f0d-4efa-a5c0-02f5375274eb

Full Demo Video (4m 35s)

https://www.youtube.com/watch?v=Z--yd8zK_Hk

## What is this?

A collection of [pi agent](https://github.com/skylar-anderson/pi) skills that let your AI agent delegate tasks to external web-based AI systems — Gemini, Grok, DeepSeek, Claude, and ChatGPT — by controlling a real browser session.

Unlike API-based integrations, these skills work with your existing logged-in browser sessions. No API keys, no billing setup, no rate limit configuration.

## How it works

```
User request
  → web-ai-router (picks the best AI for the task)
    → worker skill (navigates browser, sends prompt, waits for response)
      → returns unified JSON output
```

The agent reads the page like a human would — finding input boxes, typing prompts, waiting for responses, and extracting the reply text.

---

## Skills

| Skill | Target | Login Required | Best For |
|-------|--------|---------------|----------|
| `web-ai-router` | — | — | Auto-routes tasks to the best available AI |
| `gemini-worker` | gemini.google.com | ❌ No | General QA, translation, summarization |
| `grok-worker` | grok.com | ✅ Yes | Real-time search, breaking news, X/Twitter context |
| `deepseek-worker` | chat.deepseek.com | ✅ Yes | Deep reasoning, math, code analysis |
| `claude-worker` | claude.ai | ✅ Yes | Writing, analysis, long-form tasks |
| `chatgpt-worker` | chatgpt.com | ✅ Yes | General QA, coding, creative tasks |

---

## Requirements

- [pi agent](https://github.com/skylar-anderson/pi) v0.77.0 or later
- `local-browser-tools` skill (included in pi's default skill set)
- Chrome browser running with remote debugging enabled (managed by `browser-start.js`)
- Logged-in sessions for the AI services you want to use (except Gemini, which works as a guest)

---

## Installation

1. Clone this repository into your pi skills directory:

```bash
git clone https://github.com/Syysean/web-ai-skills ~/.pi/agent/skills/web-ai-skills
```

2. Copy each skill folder into your skills directory:

```bash
cp -r ~/.pi/agent/skills/web-ai-skills/web-ai-router ~/.pi/agent/skills/
cp -r ~/.pi/agent/skills/web-ai-skills/gemini-worker ~/.pi/agent/skills/
cp -r ~/.pi/agent/skills/web-ai-skills/grok-worker ~/.pi/agent/skills/
cp -r ~/.pi/agent/skills/web-ai-skills/deepseek-worker ~/.pi/agent/skills/
cp -r ~/.pi/agent/skills/web-ai-skills/claude-worker ~/.pi/agent/skills/
cp -r ~/.pi/agent/skills/web-ai-skills/chatgpt-worker ~/.pi/agent/skills/
```

3. Restart pi to load the new skills:

```bash
pi
```

4. Log in to the AI services you want to use in your Chrome browser. Sessions are persisted in the browser profile used by `browser-start.js`.

---

## Usage

### Auto-routing (recommended)

Let the router decide which AI to use:

```
请用外部AI帮我搜索一下今天的AI新闻
Use an external AI to analyze this code
```

### Explicit routing

Name the AI you want:

```
让 Grok 搜索一下今天发生了什么
Use DeepSeek to solve this math problem
Ask Claude to rewrite this paragraph
```

### Multi-AI comparison

```
Ask both Gemini and Grok about X and compare their answers
```

---

## Output Format

All workers return a unified JSON schema:

```json
{
  "site": "gemini | grok | deepseek | claude | chatgpt",
  "task_type": "qa | summary | translation | rewrite | extraction | comparison | brainstorming | analysis",
  "status": "success | partial | error",
  "answer": "<response text>",
  "retry_count": 0,
  "elapsed_ms": 12000,
  "error_reason": null
}
```

Followed by a short human-readable summary.

---

## Notes

- **Gemini** works without login (guest mode confirmed).
- **Grok**, **DeepSeek**, **Claude**, and **ChatGPT** require you to be logged in via Chrome before using the corresponding worker.
- If a session expires, the worker returns `error_reason: "login_required"` and falls back to `gemini-worker`.
- These skills control a real browser — they are slower than API calls but work with free-tier accounts.

---

## License

MIT
