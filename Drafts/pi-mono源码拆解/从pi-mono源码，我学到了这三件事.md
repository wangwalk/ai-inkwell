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

写一个 Agent Demo 很容易。调个 API，套个 while 循环，工具调用处理一下，跑通了截个图发出去，看起来挺厉害。

但真要把它交给用户用，你会发现问题一个接一个：用户中途改主意怎么办？对话太长撑爆 context 怎么办？工具返回了乱七八糟的东西 LLM 怎么理解？

我最近把 [pi-mono](https://github.com/badlogic/lemminx) 的源码从头拆了一遍。这是一个开源的 coding agent 框架，Claude Code 同类产品，代码写得很克制，没有过度设计。拆完之后我觉得它对上面三个问题的回答很有参考价值，分享一下。

---

## 第一件事：Loop 要跑稳，不只是跑通

我们写 Agent Loop 的时候通常是这样的：

```typescript
while (true) {
  const response = await llm.call(messages);
  if (response.stopReason === "stop") break;
  const results = await executeTools(response.toolCalls);
  messages.push(...results);
}
```

能跑，但太脆。用户在工具执行到一半的时候发了一条新消息怎么办？LLM 报错了要不要重试？用户想直接取消怎么办？

pi-mono 在这里做了两件我觉得很值得学的事。

**第一件：把"用户中途改主意"变成一个一等公民**

它在 Loop 里维护了两个队列，叫 Steering 和 FollowUp：

```typescript
// packages/agent/src/agent.ts
steer(m: AgentMessage) {
  this.steeringQueue.push(m);  // 打断当前执行
}

followUp(m: AgentMessage) {
  this.followUpQueue.push(m);  // 等 Agent 空了再说
}
```

Steering 是"现在就要插话"——每个工具执行完之后，Loop 会检查 steering 队列，如果有消息，剩下的工具调用直接跳过，LLM 重新评估。FollowUp 是"你忙完了我再说"——只有 Agent 真正空闲了才会被处理。

```typescript
// packages/agent/src/agent-loop.ts（简化）
for (let i = 0; i < toolCalls.length; i++) {
  await executeTool(toolCalls[i]);

  // 每个工具跑完都检查一次
  const steering = await getSteeringMessages();
  if (steering.length > 0) {
    // 剩下的工具：跳过
    skipRemainingToolCalls(toolCalls.slice(i + 1));
    break;
  }
}
```

这个设计让我印象深的点是：**它没有试图让 Agent 聪明地"理解"用户想打断**，而是在工程层面给了一个确定性的机制。用户发消息 → 进队列 → 下一个工具间隙处理，行为完全可预期。

**第二件：Agent 引擎和"稳定运行"的职责分开**

pi-mono 在 Agent Loop（`pi-agent-core`）之上还包了一层叫 `AgentSession`（在 `pi-coding-agent` 里），专门处理生产环境的杂活：

- API 限流了，自动重试
- context 快满了，触发压缩
- 对话历史要持久化
- 出错了要把错误信息格式化给用户看

这层不属于"Agent 逻辑"，但少了它就跑不稳。你写自己的 Agent 的时候也可以把这块单独抽出来，不要和业务逻辑混在一起，不然改起来会很痛苦。

---

## 第二件事：Context 管理，别等到撑爆了再想

Agent 和 Chatbot 有一个本质区别：一次用户请求可能触发 5 次、10 次 LLM 调用，context 涨得很快。

大部分人的处理方式是等到报错了再说，或者暴力截断最早的消息。pi-mono 有个更值得学的思路。

**把"发给 LLM 的消息"和"内部对话历史"分开**

pi-mono 里有两层消息格式：

- `AgentMessage`：Agent 内部用的，可以装任何东西
- `Message`：LLM 认识的标准格式

Loop 里全程操作 `AgentMessage[]`，只有在真正调 LLM 的那一刻，才通过 `convertToLlm` 转换：

```typescript
// packages/agent/src/agent-loop.ts（简化）
async function streamAssistantResponse(context, config) {
  // 第一步：可选裁剪，还在 AgentMessage 层
  let messages = context.messages;
  if (config.transformContext) {
    messages = await config.transformContext(messages);
  }

  // 第二步：转成 LLM 能理解的格式
  const llmMessages = await config.convertToLlm(messages);

  return await callLLM(llmMessages);
}
```

这个分层带来的好处是：**你可以在内部历史里存 LLM 不需要看到的东西**。

举个例子：用户切换了 Git 分支，你想在 UI 里展示一条"已切换到 feature-x"的通知，但这条通知对 LLM 没意义，不应该占 token。用 `CustomAgentMessages` 加一个 `notification` 类型，`convertToLlm` 里过滤掉，完事。内部历史完整，LLM 只看它需要看的。

如果你全程只维护一个 `Message[]`，这种需求要么塞进 system prompt，要么 UI 层自己维护一个平行数据结构，都很难受。

**context 压缩的触发机制**

`AgentSession` 里有一套压缩逻辑：监测当前 context 占用的 token 数，接近限制时自动找一个合适的切割点，把旧的消息总结成一段摘要，替换掉原来那些消息。

具体的切割策略我就不展开了，核心思路是：**在 context 用尽之前主动压缩，而不是被动报错**。你的 Agent 支持多长的任务，很大程度上取决于这块做得怎么样。

---

## 第三件事：工具设计要站在 LLM 视角

这块是我觉得最容易被忽视、但最影响实际效果的地方。

工具不是给人用的，是给 LLM 用的。LLM 调工具，靠的是你的描述和返回值。描述写得模糊，LLM 会调错；返回值格式乱，LLM 会误解；权限边界不清，LLM 会干出让用户意外的事。

pi-mono 的内置工具（read、edit、bash、write）有几个我觉得可以直接学走的设计原则：

**返回值要对 LLM 友好，不是对人友好**

工具报错的时候，大多数人的第一反应是抛异常或者返回一个 HTTP 风格的状态码。但 LLM 不会处理异常，它只能读文本。pi-mono 的做法是把错误信息格式化成自然语言，让 LLM 能理解并决定下一步：

```
// 不好：{"error": "ENOENT", "code": 2}
// 好：  "File not found: /path/to/file.ts. Did you mean /path/to/files.ts?"
```

LLM 看到第二种能自己决定是否重试、用别的路径，看到第一种只能懵着。

**工具的描述要说清楚边界，不只是功能**

在工具的 description 里，"能做什么"和"不应该做什么"同样重要。

比如 bash 工具，如果你只写"执行 shell 命令"，LLM 在某些情况下可能会试图用它做 `rm -rf`。加上"不用于文件删除，删除文件请用 delete 工具"，LLM 的行为会收敛很多。边界越清晰，LLM 越不容易走偏。

**可扩展，但扩展有代价**

pi-mono 有一套插件系统，允许外部注册自定义工具：

```typescript
// 一个最简单的自定义工具
api.registerTool({
  name: "search_web",
  description: "Search the web for information",
  input_schema: { /* ... */ },
  execute: async ({ query }) => {
    return await doSearch(query);
  }
});
```

但它在加载外部插件时有明确的权限检查和路径保护——不是所有东西都能注册进来，也不是注册进来就能干任何事。

这里有个值得想的问题：**工具越多，LLM 选错工具的概率越高**。内置工具应该是那些高频、通用、低风险的能力；特定场景的工具通过插件按需加载，不要一股脑全塞进 context。这个原则在自己设计工具集的时候同样适用。

---

读完这三部分，如果要我用一句话总结 pi-mono 给的答案，大概是这样的：

**Demo 验证想法，但"跑稳"、"管好 context"、"让 LLM 能用好工具"这三件事，决定你的 Agent 能不能真的交出去用。**

它不是什么高深的技术，但源码里每一个细节背后都有一个具体的生产问题。值得多读两遍。
