---
series: 深入AI-Agent-pi-mono源码拆解
title: "从 pi-mono 源码，我学到了这三件事"
date: "2026-04-09"
status: draft
tags:
  - agent
  - architecture
  - source-code
  - production
published:
  公众号:
  知乎:
  博客:
---

最近把 pi-mono 源码从头拆了一遍。这是一个开源的 coding agent 框架，Claude Code 的同类产品。

读完最大的感受：Agent Demo 好写，难的是跑稳。下面这些问题，是我拆源码之前答不上来的。

---

## 一、Loop 怎么跑稳

**用户在 Agent 执行途中改主意了，怎么办？**

大多数 Demo 里，Agent 跑起来就是个黑盒，用户只能等。pi-mono 在 Loop 里维护了两条队列：

- **Steering**：立刻插话。当前工具跑完就检查，有消息则跳过剩余工具，LLM 重新评估。
- **FollowUp**：排队等。Agent 彻底空闲了才处理。

```typescript
// packages/agent/src/agent-loop.ts
for (let i = 0; i < toolCalls.length; i++) {
  await executeTool(toolCalls[i]);

  const steering = await getSteeringMessages();
  if (steering.length > 0) {
    skipRemainingToolCalls(toolCalls.slice(i + 1));
    break;
  }
}
```

关键点：不是让 Agent "理解"用户想打断，而是工程层面的确定性机制。行为可预期。

---

**Agent 引擎和"跑稳的逻辑"该放一起吗？**

不该。pi-mono 把它们拆成两层：

- `pi-agent-core`：只管 Loop——消息循环、工具调用、停止判断
- `AgentSession`（`pi-coding-agent`）：只管稳定性——重试、context 压缩、持久化、错误格式化

Loop 里不做重试，不做压缩。AgentSession 里不管业务逻辑。

好处是：Loop 出 bug，不用看 Session 代码；Session 出问题，不影响 Loop 逻辑。混在一起的话，改一个容易踩另一个。

---

## 二、Context 怎么管

**context 快满了怎么办？**

两种常见错误做法：等报错了再处理；暴力截断最旧的消息。

pi-mono 的做法：在 `AgentSession` 里持续监测 token 用量，**接近上限前主动压缩**——找一个合适的切割点，把旧消息总结成摘要，替换掉原始内容。

```
原始历史：[msg1][msg2][msg3]...[msg50][msg51][msg52]
压缩后：  [摘要: msg1-40 的内容][msg41][msg42]...[msg52]
```

Agent 能处理多长的任务，很大程度上取决于这块做得怎么样。

---

**哪些东西该存历史，但不该发给 LLM？**

这是 pi-mono 里我觉得最值得学的设计之一。

它把消息分两层：
- `AgentMessage`：内部历史，可以存任何类型
- `Message`：LLM 格式，调 LLM 前才转换

```typescript
// 调 LLM 前的两步转换
let messages = context.messages;
if (config.transformContext) {
  messages = await config.transformContext(messages); // 裁剪
}
const llmMessages = await config.convertToLlm(messages); // 过滤+转格式
```

实际用途：用户切换了 Git 分支，UI 要显示一条通知，但这条通知对 LLM 没意义、不该占 token。用自定义消息类型存进历史，`convertToLlm` 里过滤掉，LLM 看不到，UI 能渲染。

如果全程只用一个 `Message[]`，这类需求要么塞 system prompt，要么 UI 层自己维护平行数据结构。两个都很难受。

---

## 三、工具怎么设计

**工具报错了，返回什么格式？**

不是 HTTP 状态码，不是异常，是自然语言。

```
// 别这样
{"error": "ENOENT", "code": 2}

// 这样
"File not found: /src/utils.ts. Did you mean /src/util.ts?"
```

LLM 看到第一种只能懵着；看到第二种能自己决定是否重试、换个路径试试。工具的错误信息，写给 LLM 看，不是写给开发者看。

---

**工具的 description 只写"能做什么"够吗？**

不够。"不该做什么"同样重要。

pi-mono 的 bash 工具描述里明确写了哪些操作不应该用它来做。如果只写"执行 shell 命令"，LLM 在某些情况下会用它做文件删除、系统修改。边界越清晰，行为越可预期。

---

**工具越多越好吗？**

不是。工具越多，LLM 选错的概率越高。

pi-mono 的策略：内置工具只保留高频、通用、低风险的（read、edit、bash、write）；其他能力通过插件按需加载，不全塞进 context。

```typescript
// 插件注册自定义工具，不影响内置工具集
api.registerTool({
  name: "search_web",
  description: "Search the web. Use only when information is not available locally.",
  execute: async ({ query }) => { /* ... */ }
});
```

需要什么工具，加载什么。不用的不出现在 context 里。

---

这几个问题，在写 Demo 的时候根本不会遇到。但只要你想把 Agent 交给真实用户用，迟早都得面对。pi-mono 源码值得读，不是因为它多复杂，而是因为它在这些细节上选了一个务实的解法。
