---
name: grok-worker
description: "Use this skill to send a prompt to Grok and get a response. Triggers when the user wants to query Grok, use Grok as an AI worker, or when the web-ai-router directs a task to Grok. Best for tasks requiring real-time web search, current events, or news. Requires login — session is persisted in Chrome profile."
---

## Shell Escaping Note
When passing JavaScript to browser-eval.js, always use double quotes
for the outer shell argument and single quotes inside the JavaScript.
Example: browser-eval.js "document.querySelector('.class')?.innerText"

# Grok Worker

Send a prompt to Grok via browser and return the response.

## Environment

- URL: https://grok.com
- Browser: Chrome on localhost:9222 (managed by local-browser-tools)
- Login: Required — session persisted in ~/.cache/browser-tools profile
- Tools: `browser-start.js`, `browser-nav.js`, `browser-eval.js`, `browser-screenshot.js`

---

## Input

```
prompt: string        # The text to send to Grok
task_type: string     # qa | summary | translation | rewrite | extraction | comparison | brainstorming | analysis
```

---

## Output

Always output this JSON block first:

```json
{
  "site": "grok",
  "task_type": "<task_type>",
  "status": "success | partial | error",
  "answer": "<extracted response text>",
  "retry_count": 0,
  "elapsed_ms": 0,
  "error_reason": null
}
```

Then follow with a short human-readable summary (2–3 sentences max).

---

## Workflow

### Step 1 — Start browser

```bash
node ~/.pi/agent/skills/local-browser-tools/browser-start.js
```

Expected: `✓ Chrome already running on :9222` or `✓ Chrome started on :9222`

If fails: return error status, stop.

---

### Step 2 — Navigate to Grok

```bash
node ~/.pi/agent/skills/local-browser-tools/browser-nav.js https://grok.com
```

Take a screenshot and verify:
- Input box is visible
- User avatar or username is visible in the top-right (confirms login)

If login page appears with no input box: return `error_reason: "login_required"`, stop.

---

### Step 3 — Locate input box

Grok uses a `div[contenteditable]` with `role="textbox"` and `aria-label="Ask Grok anything"`.

Check:

```javascript
!!document.querySelector('[aria-label="Ask Grok anything"]')
```

Fallback selectors in order:
1. `[aria-label="Ask Grok anything"]`
2. `[contenteditable="true"][role="textbox"]`
3. `div[contenteditable]`

If no selector works after 3 attempts: trigger retry.

---

### Step 4 — Insert prompt

```javascript
(function() {
  const input = document.querySelector('[aria-label="Ask Grok anything"]')
    || document.querySelector('[contenteditable="true"][role="textbox"]')
    || document.querySelector('div[contenteditable]');
  if (!input) return "NOT_FOUND";
  input.focus();
  input.textContent = "";
  input.textContent = PROMPT_TEXT;
  input.dispatchEvent(new Event("input", { bubbles: true }));
  return input.textContent;
})()
```

Verify returned value matches the intended prompt. If mismatch: retry.

Take a screenshot to confirm text appears in the input box.

---

### Step 5 — Send

Grok has no send button with aria-label. Use Enter key to send:

```javascript
document.querySelector('[aria-label="Ask Grok anything"]')
  ?.dispatchEvent(new KeyboardEvent("keydown", { key: "Enter", bubbles: true, cancelable: true }))
```

After sending, take a screenshot to confirm the message bubble appears in the chat.

---

### Step 6 — Wait for response

Poll every 3 seconds, maximum 120 seconds total.
Do NOT use fixed sleep.

Spinner selectors are unreliable. Use response length stability as the primary completion signal:

1. After sending, start polling response text length
2. Take a screenshot on the first poll where length > 0 — save as "thinking" screenshot
3. Treat response as complete when length is non-zero and unchanged across 2 consecutive polls — save as "response complete" screenshot

Response extraction for polling:

​```javascript
(function() {
  const els = document.querySelectorAll('.response-content-markdown');
  if (els.length > 0) return els[els.length - 1].innerText?.trim()?.length || 0;
  return 0;
})()
​```

If 120 seconds pass with no stable response: trigger retry.

---

### Step 7 — Extract response

​```javascript
(function() {
  const els = document.querySelectorAll('.response-content-markdown');
  if (els.length > 0) return els[els.length - 1].innerText?.trim();
  return "EXTRACTION_FAILED";
})()
​```

Validate the result:
- Must contain text
- Must not equal the original prompt
- Must not be UI chrome text (e.g. "Ask Grok anything", "Sign in")

If validation fails: trigger retry.

---

## Retry Policy

- Maximum retries: 2
- On each retry: re-navigate to `https://grok.com` before retrying
- After 2nd failure: return `status: "error"` with `error_reason` explaining which step failed

---

## Selector Change Resilience

If a selector stops working:
1. Use `browser-eval.js` to list all `[contenteditable]` and `button` and `[role="button"]` elements
2. Identify the correct element by `aria-label`, `role`, `placeholder`, or visible text
3. Use the working selector for this session

---

## Example Output

```json
{
  "site": "grok",
  "task_type": "qa",
  "status": "success",
  "answer": "Here is the latest news...",
  "retry_count": 0,
  "elapsed_ms": 22000,
  "error_reason": null
}
```