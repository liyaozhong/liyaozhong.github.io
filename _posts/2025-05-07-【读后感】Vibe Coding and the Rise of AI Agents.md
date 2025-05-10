---
layout: post
title: "【读后感】Vibe Coding and the Rise of AI Agents"
subtitle: "核心技能：沟通、协调、逻辑思维和任务分解"
tags: [Reading, Vibe Coding]
comments: false
---
[原文](https://thedataexchange.media/vibe-coding-chop-steve-yegge/)

### Vibe Coding 的定义与演变

- Vibe coding 被描述为超越传统代码补全的范式，传统代码补全如今被视为过时。
- 代理编程（agent programming）正在以指数级速度超越聊天编程（chat programming），提供更优的结果。
- 相关参考：[Gradient Flow 关于 vibe coding 和 CHOP 的解释](https://gradientflow.com/vibe-coding-and-chop-what-you-need-to-know/)。

### 开发者角色的转变

- 开发者从逐行编写代码转向高层次的协调角色，负责管理 AI 生成的代码，同时保持对代码质量的责任。
- 这种转变对许多开发者来说具有心理挑战，因为他们的职业身份往往建立在传统编码技能之上。

### 对 AI 辅助编码的信任问题

- 信任被认为是采用 AI 工具的主要障碍。
- LLM 有 20% 的幻觉率（hallucination rate），这意味着开发者必须验证 AI 生成的代码，将开发过程转变为验证过程。

### 未来所需的核心技能

- 开发者需要深入了解 AI 工程，文章推荐 Chip Huyen 的《AI 工程》作为学习资源：[AI 工程](https://www.oreilly.com/library/view/ai-engineering/9781098166298/)。
- 关键技能包括：
    - Vibe coding 和提示工程（prompt engineering）。
    - 人文学科技能，如沟通、协调、逻辑思维和任务分解。

### 编码代理与聊天编程的对比

- 编码代理在持续循环中操作，比聊天编程快 5 倍，而聊天编程本身比手动编码强 5 倍。
- 具体例子：Claude Code 可处理 JIRA 票据、代码库分析、测试和文档，生产力相当于每小时 10 美元。

### 大型项目中的挑战

- LLM 在没有模型上下文协议（MCP）服务器的情况下难以处理上下文，MCP 被描述为“新的 HTTP”。
- 当前，开发者必须手动提供上下文，这效率低下，公司需要投资构建 MCP 服务器以改善效率。