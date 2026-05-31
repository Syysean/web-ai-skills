# Changelog

All notable changes to this project will be documented in this file.

---

## [1.0.0] - 2026-05-31

### Added

- `web-ai-router` — auto-routes user tasks to the best available AI worker based on task type
- `worker-contract.md` — unified input/output schema shared by all workers
- `gemini-worker` — sends prompts to Gemini via browser; guest mode confirmed working without login
- `grok-worker` — sends prompts to Grok via browser; supports real-time web search
- `deepseek-worker` — sends prompts to DeepSeek via browser; strong reasoning and math capability
- `claude-worker` — sends prompts to Claude.ai via browser; uses ProseMirror-compatible input method
- `chatgpt-worker` — sends prompts to ChatGPT via browser; uses ProseMirror div input, not the fallback textarea

### Technical Notes

- All workers use response length stability polling instead of spinner detection (spinner selectors proved unreliable across all tested sites)
- DeepSeek requires `nativeInputValueSetter` for React state update; direct `textarea.value` assignment silently fails
- ChatGPT's real input element is `div.ProseMirror`, not the `textarea.wcDTda_fallbackTextarea` accessibility fallback — writing to the textarea causes the send button to trigger voice mode instead of sending
- Grok send is triggered via `Enter` keydown on the contenteditable input; no reliable send button aria-label exists
- Gemini send button must be located via `Array.from(buttons).find()` with aria-label matching; Angular dynamic class names are unreliable as selectors