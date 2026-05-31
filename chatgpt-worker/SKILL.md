---
name: chatgpt-worker
description: "Use this skill to send a prompt to ChatGPT and get a response. Triggers when the user wants to query ChatGPT, use ChatGPT as an external AI worker, or when the web-ai-router directs a task to ChatGPT. Best for general QA, writing, coding, and creative tasks. Requires login — session is persisted in Chrome profile."
---

# ChatGPT Worker

Send a prompt to ChatGPT via browser and return the response.

## Environment

- URL: https://chatgpt.com
- Browser: Chrome on localhost:9222 (managed by local-browser-tools)
- Login: Required — session persisted in ~/.cache/browser-tools profile
- Tools: `browser-start.js`, `browser-nav.js`, `browser-eval.js`, `browser-screenshot.js`

---

## Input

```
prompt: string        # The text to send to ChatGPT
task_type: string     # qa | summary | translation | rewrite | extraction | comparison | brainstorming | analysis
```

---

## Output

Always output this JSON block first:

```json
{
  "site": "chatgpt",
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

## Critical Architecture Note

ChatGPT uses **ProseMirror** as its rich text editor. The real input element is a `div.ProseMirror`, NOT the `<textarea>` on the page. The textarea (`wcDTda_fallbackTextarea`) is a hidden accessibility fallback that React does not monitor for input. Writing to the textarea will never trigger the send button — it will always launch voice mode instead.

---

## Workflow

### Step 1 — Start browser

```bash
node ~/.pi/agent/skills/local-browser-tools/browser-start.js
```

Expected: `✓ Chrome already running on :9222` or `✓ Chrome started on :9222`

If fails: return error status, stop.

---

### Step 2 — Navigate to ChatGPT

```bash
node ~/.pi/agent/skills/local-browser-tools/browser-nav.js https://chatgpt.com
```

Take a screenshot and verify:
- Input area is visible
- Not showing a login wall

If login page appears: return `error_reason: "login_required"`, stop.

---

### Step 3 — Locate input box

ChatGPT's real input is a ProseMirror div, NOT a textarea.

Check:

```javascript
!!document.querySelector('div.ProseMirror[contenteditable="true"]')
```

Fallback selectors in order:
1. `div.ProseMirror[contenteditable="true"]`
2. `div[role="textbox"][aria-label="Chat with ChatGPT"]`
3. `div[contenteditable="true"][role="textbox"]`

DO NOT use `textarea` — it is the wrong element.

If no selector works after 3 attempts: trigger retry.

---

### Step 4 — Insert prompt

Use `execCommand("insertText")` — this is the correct method for ProseMirror editors. Do NOT set `innerHTML` or `textContent` directly as ProseMirror will not register the change.

```javascript
(function() {
  const input = document.querySelector('div.ProseMirror[contenteditable="true"]')
    || document.querySelector('div[role="textbox"][aria-label="Chat with ChatGPT"]')
    || document.querySelector('div[contenteditable="true"][role="textbox"]');
  if (!input) return "NOT_FOUND";
  input.focus();
  input.innerHTML = "";
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

Take a screenshot to confirm text appears in the input area.

---

### Step 5 — Send

After Step 4 succeeds, the `composer-submit-button-color` button will behave as a send button. Click it:

```javascript
document.querySelector('button.composer-submit-button-color')?.click()
```

After clicking, take a screenshot to confirm:
- The message bubble appears in the chat
- The input area is cleared

If the input area is NOT cleared: Step 4 likely wrote to the wrong element. Re-check Step 3 selector and retry from Step 4.

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
  const msgs = document.querySelectorAll('[data-message-author-role="assistant"]');
  if (msgs.length > 0) return msgs[msgs.length - 1].innerText?.trim()?.length || 0;
  return 0;
})()
```

If 120 seconds pass with no stable response: trigger retry.

---

### Step 7 — Extract response

```javascript
(function() {
  const msgs = document.querySelectorAll('[data-message-author-role="assistant"]');
  if (msgs.length > 0) {
    return msgs[msgs.length - 1].innerText?.trim();
  }
  const candidates = document.querySelectorAll('[class*="markdown"], [class*="prose"]');
  if (candidates.length > 0) {
    return candidates[candidates.length - 1].innerText?.trim();
  }
  return "EXTRACTION_FAILED";
})()
```

Validate the result:
- Must contain text
- Must not equal the original prompt
- Must not be UI chrome text (e.g. "Ask anything", "ChatGPT can make mistakes")

If validation fails: trigger retry.

---

## Retry Policy

- Maximum retries: 2
- On each retry: re-navigate to `https://chatgpt.com` before retrying
- After 2nd failure: return `status: "error"` with `error_reason` explaining which step failed

---

## Selector Change Resilience

If a selector stops working:
1. Use `browser-eval.js` to list all `[contenteditable]` elements: `Array.from(document.querySelectorAll('[contenteditable]')).map(e => ({tag: e.tagName, role: e.getAttribute('role'), aria: e.getAttribute('aria-label'), cls: String(e.className).slice(0, 80)}))`
2. Identify the ProseMirror editor by its className or role="textbox"
3. Never use the `textarea` element — it is always the wrong target

---

## Example Output

```json
{
  "site": "chatgpt",
  "task_type": "qa",
  "status": "success",
  "answer": "TEST-CHATGPT",
  "retry_count": 0,
  "elapsed_ms": 14000,
  "error_reason": null
}
```