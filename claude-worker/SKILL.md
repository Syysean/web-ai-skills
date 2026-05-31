---
name: claude-worker
description: "Use this skill to send a prompt to Claude.ai and get a response. Triggers when the user wants to query Claude.ai via browser, use Claude as an external AI worker, or when the web-ai-router directs a task to Claude. Best for general QA, writing, analysis, and coding tasks. Requires login — session is persisted in Chrome profile."
---

# Claude Worker

Send a prompt to Claude.ai via browser and return the response.

## Environment

- URL: https://claude.ai
- Browser: Chrome on localhost:9222 (managed by local-browser-tools)
- Login: Required — session persisted in ~/.cache/browser-tools profile
- Tools: `browser-start.js`, `browser-nav.js`, `browser-eval.js`, `browser-screenshot.js`

---

## Input

```
prompt: string        # The text to send to Claude
task_type: string     # qa | summary | translation | rewrite | extraction | comparison | brainstorming | analysis
```

---

## Output

Always output this JSON block first:

```json
{
  "site": "claude",
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

### Step 2 — Navigate to Claude

```bash
node ~/.pi/agent/skills/local-browser-tools/browser-nav.js https://claude.ai
```

Take a screenshot and verify:
- Input box is visible
- Page title contains "Claude"
- Not showing a login wall

If login page appears with no input box: return `error_reason: "login_required"`, stop.

---

### Step 3 — Locate input box

Claude uses a `div[contenteditable]` with `role="textbox"` and `aria-label="Write your prompt to Claude"`.

Check:

```javascript
!!document.querySelector('[aria-label="Write your prompt to Claude"]')
```

Fallback selectors in order:
1. `[aria-label="Write your prompt to Claude"]`
2. `[contenteditable="true"][role="textbox"]`
3. `div[contenteditable]`

If no selector works after 3 attempts: trigger retry.

---

### Step 4 — Insert prompt

Claude uses a contenteditable div. Use `execCommand` to insert text — do NOT set `textContent` directly as it bypasses the editor state:

```javascript
(function() {
  const input = document.querySelector('[aria-label="Write your prompt to Claude"]')
    || document.querySelector('[contenteditable="true"][role="textbox"]')
    || document.querySelector('div[contenteditable]');
  if (!input) return "NOT_FOUND";
  input.focus();
  input.innerHTML = "";
  input.dispatchEvent(new Event("input", { bubbles: true }));
  const sel = window.getSelection();
  const range = document.createRange();
  range.selectNodeContents(input);
  sel.removeAllRanges();
  sel.addRange(range);
  document.execCommand("insertText", false, PROMPT_TEXT);
  return input.textContent;
})()
```

Verify returned value matches the intended prompt. If mismatch: retry.

Take a screenshot to confirm text appears in the input box.

---

### Step 5 — Send

Claude's send button uses `data-testid`:

```javascript
(document.querySelector('button[data-testid*="send"]')
  || document.querySelector('button[aria-label*="Send"]'))
  ?.click()
```

After clicking, take a screenshot to confirm the message bubble appears in the chat.

---

### Step 6 — Wait for response

Poll every 3 seconds, maximum 120 seconds total.
Do NOT use fixed sleep.

Use response length stability as the primary completion signal:

1. After sending, start polling response text length
2. Take a screenshot on the first poll where length > 0 — save as "thinking" screenshot
3. Treat response as complete when length is non-zero and unchanged across 2 consecutive polls — save as "response complete" screenshot

Response extraction for polling:

```javascript
(function() {
  const msgs = document.querySelectorAll('div.font-claude-response');
  if (msgs.length > 0) return msgs[msgs.length - 1].innerText?.trim()?.length || 0;
  return 0;
})()
```

If 120 seconds pass with no stable response: trigger retry.

---

### Step 7 — Extract response

```javascript
(function() {
  const responses = document.querySelectorAll('div.font-claude-response');
  if (responses.length > 0) {
    return responses[responses.length - 1].innerText?.trim();
  }
  return "EXTRACTION_FAILED";
})()
```

Validate the result:
- Must contain text
- Must not equal the original prompt
- Must not be UI chrome text (e.g. "Write your prompt to Claude", "New chat")

If validation fails: trigger retry.

---

## Retry Policy

- Maximum retries: 2
- On each retry: re-navigate to `https://claude.ai` before retrying
- After 2nd failure: return `status: "error"` with `error_reason` explaining which step failed

---

## Selector Change Resilience

If a selector stops working:
1. Use `browser-eval.js` to list all `[contenteditable]` and `button[data-testid]` elements
2. Identify the correct element by `aria-label`, `role`, or `data-testid`
3. Use the working selector for this session

---

## Example Output

```json
{
  "site": "claude",
  "task_type": "qa",
  "status": "success",
  "answer": "TEST-OK",
  "retry_count": 0,
  "elapsed_ms": 15000,
  "error_reason": null
}
```