# web-ai-skills

**通过浏览器自动化向网页版 AI 发送提示词的 Pi Agent Skills 集合 — 无需 API Key。**

[English](./README.md)

---

## 这是什么？

这是一套 [pi agent](https://github.com/skylar-anderson/pi) skills，让你的 AI 代理能够把任务委托给外部网页版 AI 系统 — Gemini、Grok、DeepSeek、Claude 和 ChatGPT — 通过控制真实的浏览器会话来实现。

与 API 集成不同，这些 skills 使用你已经登录的浏览器会话，不需要 API Key，不需要配置账单，不需要管理速率限制。

## 工作原理

```
用户请求
  → web-ai-router（选择最合适的 AI）
    → worker skill（导航浏览器、输入提示词、等待回复、提取结果）
      → 返回统一的 JSON 输出
```

Agent 像人一样操作页面 — 找到输入框、输入提示词、等待回复、提取回复内容。

---

## Skills 列表

| Skill | 目标网站 | 需要登录 | 最适合的任务 |
|-------|--------|---------|------------|
| `web-ai-router` | — | — | 自动路由到最合适的 AI |
| `gemini-worker` | gemini.google.com | ❌ 不需要 | 通用问答、翻译、摘要 |
| `grok-worker` | grok.com | ✅ 需要 | 实时搜索、突发新闻、X/Twitter 内容 |
| `deepseek-worker` | chat.deepseek.com | ✅ 需要 | 深度推理、数学、代码分析 |
| `claude-worker` | claude.ai | ✅ 需要 | 写作、分析、长文任务 |
| `chatgpt-worker` | chatgpt.com | ✅ 需要 | 通用问答、编程、创意任务 |

---

## 环境要求

- [pi agent](https://github.com/skylar-anderson/pi) v0.77.0 或更高版本
- `local-browser-tools` skill（pi 默认 skill 集中已包含）
- 启用了远程调试的 Chrome 浏览器（由 `browser-start.js` 管理）
- 你想使用的 AI 服务的已登录会话（Gemini 游客模式可用，无需登录）

---

## 安装

1. 克隆本仓库到你的 pi skills 目录：

```bash
git clone https://github.com/Syysean/web-ai-skills ~/.pi/agent/skills/web-ai-skills
```

2. 将每个 skill 文件夹复制到 skills 目录：

```bash
cp -r ~/.pi/agent/skills/web-ai-skills/web-ai-router ~/.pi/agent/skills/
cp -r ~/.pi/agent/skills/web-ai-skills/gemini-worker ~/.pi/agent/skills/
cp -r ~/.pi/agent/skills/web-ai-skills/grok-worker ~/.pi/agent/skills/
cp -r ~/.pi/agent/skills/web-ai-skills/deepseek-worker ~/.pi/agent/skills/
cp -r ~/.pi/agent/skills/web-ai-skills/claude-worker ~/.pi/agent/skills/
cp -r ~/.pi/agent/skills/web-ai-skills/chatgpt-worker ~/.pi/agent/skills/
```

3. 重启 pi 加载新 skills：

```bash
pi
```

4. 在 Chrome 浏览器中登录你想使用的 AI 服务。会话会保存在 `browser-start.js` 使用的浏览器 profile 中。

---

## 使用方式

### 自动路由（推荐）

让 router 自动选择最合适的 AI：

```
请用外部AI帮我搜索一下今天的AI新闻
用外部AI分析一下这段代码
```

### 指定 AI

直接点名你想用的 AI：

```
让 Grok 搜索一下今天发生了什么
用 DeepSeek 解这道数学题
让 Claude 帮我改写这段文字
```

### 多 AI 对比

```
同时问 Gemini 和 Grok 关于 X 的问题，对比他们的回答
```

---

## 输出格式

所有 worker 返回统一的 JSON 格式：

```json
{
  "site": "gemini | grok | deepseek | claude | chatgpt",
  "task_type": "qa | summary | translation | rewrite | extraction | comparison | brainstorming | analysis",
  "status": "success | partial | error",
  "answer": "<回复内容>",
  "retry_count": 0,
  "elapsed_ms": 12000,
  "error_reason": null
}
```

后面跟一段简短的人类可读摘要。

---

## 注意事项

- **Gemini** 无需登录（游客模式已验证可用）。
- **Grok**、**DeepSeek**、**Claude** 和 **ChatGPT** 在使用对应 worker 之前需要先在 Chrome 中完成登录。
- 如果会话过期，worker 会返回 `error_reason: "login_required"` 并自动回退到 `gemini-worker`。
- 这些 skills 控制的是真实浏览器，速度比直接调用 API 慢，但适用于免费账户。

---

## 开源协议

MIT