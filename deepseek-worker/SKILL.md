---
name: deepseek-worker
description: "Use this skill to send a prompt to DeepSeek and get a response. Triggers when the user wants to query DeepSeek, use DeepSeek as an AI worker, or when the web-ai-router directs a task to DeepSeek. Best for deep reasoning, math, code analysis, and complex multi-step problems. Requires login — session is persisted in Chrome profile."
---

# DeepSeek Worker

Send a prompt to DeepSeek via browser and return the response.

## Environment

- URL: https://chat.deepseek.com
- Browser: Chrome on localhost:9222 (managed by local-browser-tools)
- Login: Required — session persisted in ~/.cache/browser-tools profile
- Tools: `browser-start.js`, `browser-nav.js`, `browser-eval.js`, `browser-screenshot.js`

---

## Input

```
prompt: string        # The text to send to DeepSeek
task_type: string     # qa | summary | translation | rewrite | extraction | comparison | brainstorming | analysis
```

---

## Output

Always output this JSON block first:

```json
{
  "site": "deepseek",
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

### Step 2 — Navigate to DeepSeek

```bash
node ~/.pi/agent/skills/local-browser-tools/browser-nav.js https://chat.deepseek.com
```

Take a screenshot and verify:
- Textarea input box is visible with placeholder "Message DeepSeek"
- Not showing a login form

If login page appears: return `error_reason: "login_required"`, stop.

---

### Step 3 — Locate input box

DeepSeek uses a `<textarea>` with `placeholder="Message DeepSeek"`. Note: no aria-label.

Check:

```javascript
!!document.querySelector('textarea[placeholder="Message DeepSeek"]')
```

Fallback selectors in order:
1. `textarea[placeholder="Message DeepSeek"]`
2. `textarea`

If no selector works after 3 attempts: trigger retry.

---

### Step 4 — Insert prompt

DeepSeek uses React — setting textarea.value directly does NOT update React's
internal state and will cause "Message is empty" on send. Must use nativeInputValueSetter:

​```javascript
(function() {
  const input = document.querySelector('textarea[placeholder="Message DeepSeek"]')
    || document.querySelector('textarea');
  if (!input) return "NOT_FOUND";
  input.focus();
  const nativeInputValueSetter = Object.getOwnPropertyDescriptor(
    window.HTMLTextAreaElement.prototype, 'value'
  ).set;
  nativeInputValueSetter.call(input, "");
  nativeInputValueSetter.call(input, PROMPT_TEXT);
  input.dispatchEvent(new Event("input", { bubbles: true }));
  return input.value;
})()
​```

Verify returned value matches the intended prompt. If mismatch: retry.

Take a screenshot to confirm text appears in the textarea.

---

### Step 5 — Send

DeepSeek's send button is a div with role="button", NOT a native button element.
Ctrl+Enter adds a newline — do NOT use it.

Click the send button:
​```javascript
Array.from(document.querySelectorAll('[role="button"]'))
  .filter(b => {
    const cls = typeof b.className === 'string' ? b.className : '';
    return cls.includes('ds-icon-button') && !cls.includes('ds-icon-button--disabled');
  })
  .pop()
  ?.click()
​```

After sending, take a screenshot to confirm the message bubble appears.

---

### Step 6 — Wait for response

Poll every 3 seconds, maximum 120 seconds total.
Do NOT use fixed sleep.

Spinner selectors are unreliable. Use response length stability as the primary completion signal:

1. After sending, start polling response text length
2. Take a screenshot on the first poll where length > 0 — save as "thinking" screenshot
3. Treat response as complete when length is non-zero and unchanged across 2 consecutive polls — save as "response complete" screenshot

Response extraction for polling:

```javascript
(function() {
  const msgs = document.querySelectorAll('[class*="message"], [class*="ds-markdown"], [class*="chat"]');
  if (msgs.length > 0) return msgs[msgs.length - 1].innerText?.trim()?.length || 0;
  return 0;
})()
```

If 120 seconds pass with no stable response: trigger retry.

---

### Step 7 — Extract response

​```javascript
(function() {
  const direct = document.querySelector('div.ds-markdown.ds-assistant-message-main-content');
  if (direct) return direct.innerText?.trim();
  const para = document.querySelector('p.ds-markdown-paragraph');
  if (para) return para.innerText?.trim();
  return "EXTRACTION_FAILED";
})()
​```

Validate the result:
- Must contain text
- Must not equal the original prompt
- Must not be UI chrome text (e.g. "Message DeepSeek", "DeepThink", "Search")

If validation fails: trigger retry.

---

## Retry Policy

- Maximum retries: 2
- On each retry: re-navigate to `https://chat.deepseek.com` before retrying
- After 2nd failure: return `status: "error"` with `error_reason` explaining which step failed

---

## Selector Change Resilience

DeepSeek uses hashed class names (e.g. `_27c9245`, `c3ecdb44`) that change between deployments. Do NOT rely on these. If selectors stop working:
1. Use `browser-eval.js` to find all `textarea`, `[role="button"]`, and elements with visible text
2. Identify by placeholder text, role, or visible label
3. Use the working selector for this session

---

## Example Output

```json
{
  "site": "deepseek",
  "task_type": "analysis",
  "status": "success",
  "answer": "Here is my analysis...",
  "retry_count": 0,
  "elapsed_ms": 35000,
  "error_reason": null
}
```