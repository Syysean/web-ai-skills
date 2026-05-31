---
name: gemini-worker
description: "Use this skill to send a prompt to Gemini and get a response. Triggers when the user wants to query Gemini, use Gemini as an AI worker, or when the web-ai-router directs a task to Gemini. Handles navigation, input, waiting, and response extraction automatically. No login required."
---

# Gemini Worker

Send a prompt to Gemini via browser and return the response.

## Environment

- Browser: Chrome on localhost:9222 (managed by local-browser-tools)
- Login: Not required — Gemini guest mode is confirmed working
- Tools: `browser-start.js`, `browser-nav.js`, `browser-eval.js`, `browser-screenshot.js`

---

## Input

```
prompt: string        # The text to send to Gemini
task_type: string     # qa | summary | translation | rewrite | extraction | comparison | brainstorming | analysis
```

---

## Output

Always output this JSON block first:

```json
{
  "site": "gemini",
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

### Step 2 — Navigate to Gemini

```bash
node ~/.pi/agent/skills/local-browser-tools/browser-nav.js https://gemini.google.com/app
```

After navigation, take a screenshot and verify:
- Input box is visible
- Page is NOT a login wall (a "Sign in" button in the top-right corner is normal and acceptable — guest mode works)

If a full-page login form appears with no input box: return error status, stop.

---

### Step 3 — Locate input box

Try selectors in this order until one works:

1. `[aria-label="Enter a prompt for Gemini"]`
2. `[contenteditable="true"][role="textbox"]`
3. `div.ql-editor`

Use `browser-eval.js` to check:

```javascript
!!document.querySelector('[aria-label="Enter a prompt for Gemini"]')
```

If no selector works after 3 attempts: trigger retry.

---

### Step 4 — Insert prompt

```javascript
(function() {
  const input = document.querySelector('[aria-label="Enter a prompt for Gemini"]')
    || document.querySelector('[contenteditable="true"][role="textbox"]')
    || document.querySelector('div.ql-editor');
  if (!input) return "NOT_FOUND";
  input.focus();
  input.textContent = "";
  input.textContent = PROMPT_TEXT;
  input.dispatchEvent(new Event("input", { bubbles: true }));
  return input.textContent;
})()
```

Verify returned value matches the intended prompt. If mismatch: retry.

---

### Step 5 — Send

Try send button selectors in this order — use aria-label only, not class
(Angular dynamic classes like ng-tns-* are unreliable):

1. `Array.from(document.querySelectorAll('button')).find(b => b.getAttribute('aria-label') === 'Send message')?.click()`
2. `Array.from(document.querySelectorAll('button')).find(b => b.getAttribute('aria-label')?.includes('Send'))?.click()`

If both selectors return undefined, fallback to pressing Enter on the focused input element:
`document.querySelector('[contenteditable]')?.dispatchEvent(new KeyboardEvent('keydown', { key: 'Enter', bubbles: true, cancelable: true }))`

After sending, take a screenshot to confirm the message appears in the chat.

---

### Step 6 — Wait for response

Poll every 3 seconds, maximum 120 seconds total.
Do NOT use fixed sleep.

Spinner selectors are unreliable — both match static elements on the page.
Use response length stability as the primary completion signal:

1. After sending, start polling `document.querySelector('model-response')?.innerText?.trim()?.length`
2. Take a screenshot on the first poll where length > 0 — save as "thinking" screenshot
3. Treat response as complete when length is non-zero and unchanged across 2 consecutive polls — save as "response complete" screenshot

If 120 seconds pass with no stable response: trigger retry.

---

### Step 7 — Extract response

```javascript
(function() {
  // Get all Gemini response containers
  const responseEls = document.querySelectorAll('model-response, .model-response-text, [data-response-id]');
  if (responseEls.length > 0) {
    return responseEls[responseEls.length - 1].innerText?.trim();
  }
  // Fallback: find last message block that is not the user bubble
  const allMessages = document.querySelectorAll('.conversation-container p, .response-content p');
  if (allMessages.length > 0) {
    return Array.from(allMessages).map(el => el.innerText).join('\n').trim();
  }
  return "EXTRACTION_FAILED";
})()
```

Validate the result:
- Must contain text
- Must not equal the original prompt
- Must not be UI chrome text (e.g. "Ask Gemini", "Sign in", "Gemini is AI")

If validation fails: trigger retry.

---

## Retry Policy

- Maximum retries: 2
- On each retry: re-navigate to `https://gemini.google.com/app` before retrying
- After 2nd failure: return `status: "error"` with `error_reason` explaining which step failed

---

## Decision Guide for the Agent

```
Screenshot shows spinner still spinning → keep waiting, do not extract yet
Screenshot shows "Sign in" button only (no input box) → full login wall, return error
Screenshot shows input box + "Sign in" button → guest mode, proceed normally  
Extracted text is empty or equals prompt → extraction failed, retry
```

---

## Selector Change Resilience

Google frequently updates Gemini's DOM. If a selector stops working:

1. Use `browser-eval.js` to list all `[contenteditable]` and `button` elements on the page
2. Identify the correct element by its `aria-label`, `role`, or visible text
3. Use the working selector for this session
4. Note in the output which fallback selector was used

---

## Example Output

```json
{
  "site": "gemini",
  "task_type": "qa",
  "status": "success",
  "answer": "TEST-123",
  "retry_count": 0,
  "elapsed_ms": 18400,
  "error_reason": null
}
```

**Summary:** Gemini responded successfully in 18.4 seconds. Answer: "TEST-123".