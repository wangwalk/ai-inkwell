---
series: 深入AI-Agent-pi-mono源码拆解
title: "Agent 引擎：循环的心跳"
date: "2026-03-28"
status: draft
tags:
  - agent
  - architecture
  - source-code
  - agent-loop
  - tool-calling
published:
  公众号:
  知乎:
  博客:
---

用户问一次，LLM API 可能被调了 5 次。

这就是 Agent 和 Chatbot 的本质区别。Chatbot 是一问一答——你发一条消息，LLM 回一条消息，结束。Agent 是一个循环——你发一条消息，LLM 回复说"我需要先读个文件"，读完了又说"我还得改一下"，改完了又说"跑个测试确认下"……直到它自己判断任务完成，才停下来。

这个循环就是 Agent Loop，整个 Agent 的心跳。上一篇我们看了 pi-mono 的全景图，知道 pi-agent-core 是中间那层引擎。今天我们把引擎盖打开，看看这颗心脏怎么跳。

## runLoop：双层 while 循环

Agent Loop 的核心在 `runLoop` 函数里。我把它简化到骨架：

```typescript
// pi-agent-core/src/agent-loop.ts（简化）
async function runLoop(currentContext, newMessages, config, signal, stream) {
  let pendingMessages = (await config.getSteeringMessages?.()) || [];

  // 外层循环：处理 follow-up 消息
  while (true) {
    let hasMoreToolCalls = true;

    // 内层循环：处理工具调用 + steering 消息
    while (hasMoreToolCalls || pendingMessages.length > 0) {
      // 1. 注入 pending 消息（steering 或 follow-up）
      if (pendingMessages.length > 0) {
        for (const message of pendingMessages) {
          currentContext.messages.push(message);
          newMessages.push(message);
        }
        pendingMessages = [];
      }

      // 2. 调 LLM，拿到 assistant 回复
      const message = await streamAssistantResponse(currentContext, config, signal, stream);
      newMessages.push(message);

      // 3. 如果出错或被中止，直接退出
      if (message.stopReason === "error" || message.stopReason === "aborted") {
        stream.push({ type: "agent_end", messages: newMessages });
        return;
      }

      // 4. 有工具调用就执行
      const toolCalls = message.content.filter((c) => c.type === "toolCall");
      hasMoreToolCalls = toolCalls.length > 0;

      if (hasMoreToolCalls) {
        const toolExecution = await executeToolCalls(/* ... */);
        // 工具结果塞回上下文
        for (const result of toolExecution.toolResults) {
          currentContext.messages.push(result);
          newMessages.push(result);
        }
      }

      // 5. 检查有没有新的 steering 消息
      pendingMessages = (await config.getSteeringMessages?.()) || [];
    }

    // 内层循环结束，检查 follow-up
    const followUpMessages = (await config.getFollowUpMessages?.()) || [];
    if (followUpMessages.length > 0) {
      pendingMessages = followUpMessages;
      continue; // 回到外层循环
    }

    break; // 没有 follow-up，真正结束
  }

  stream.push({ type: "agent_end", messages: newMessages });
}
```

一次典型的执行路径：用户说"把错误处理加上" → LLM 说要先读文件（tool_use）→ 读文件 → LLM 说要改代码（tool_use）→ 改代码 → LLM 说要跑测试（tool_use）→ 跑测试 → LLM 说搞定了（end_turn）→ 内层循环退出 → 没有 follow-up → 外层循环也退出。

5 次 LLM 调用，用户只按了一次回车。

## stopReason：什么时候停？

Agent 不会无限转，它根据 LLM 返回的 stopReason 决定下一步。源码里处理得很直接：

```typescript
// pi-agent-core/src/agent-loop.ts
if (message.stopReason === "error" || message.stopReason === "aborted") {
  stream.push({ type: "turn_end", message, toolResults: [] });
  stream.push({ type: "agent_end", messages: newMessages });
  stream.end(newMessages);
  return;
}

const toolCalls = message.content.filter((c) => c.type === "toolCall");
hasMoreToolCalls = toolCalls.length > 0;
```

整理成表：

| stopReason | 含义 | 循环行为 |
|-----------|------|---------|
| endTurn | LLM 认为任务完成 | 退出内层循环，检查 follow-up |
| toolUse | LLM 要调工具 | 执行工具，结果塞回，继续循环 |
| error | API 报错 | 立即退出整个循环 |
| aborted | 用户主动取消 | 立即退出整个循环 |

注意，没有一个显式的状态机或 switch-case 来路由这四种状态。它完全靠循环条件自然处理：`hasMoreToolCalls` 为 false 时内层循环退出（对应 endTurn），为 true 时继续（对应 toolUse），error/aborted 直接 return。

这是一种隐式状态机——状态转换嵌在控制流里，而不是用一个 `state` 变量显式管理。代码更短，但你得通读整个循环才能理解所有可能的路径。

## 两层消息抽象：AgentMessage vs Message

上一篇的思考题问了一个问题：为什么 Agent 内部的消息格式和 LLM 的消息格式要解耦？答案就在这个循环里。

```typescript
// pi-agent-core/src/types.ts
export type AgentMessage = Message | CustomAgentMessages[keyof CustomAgentMessages];

export interface CustomAgentMessages {
  // 空的——应用通过 declaration merging 扩展
}
```

Agent Loop 内部全程操作的是 `AgentMessage[]`。只有在调 LLM 的那一刻，才通过 `convertToLlm` 转成 `Message[]`：

```typescript
// pi-agent-core/src/agent-loop.ts
async function streamAssistantResponse(context, config, signal, stream) {
  // 第一步：AgentMessage[] → AgentMessage[]（可选裁剪）
  let messages = context.messages;
  if (config.transformContext) {
    messages = await config.transformContext(messages, signal);
  }

  // 第二步：AgentMessage[] → Message[]（必须）
  const llmMessages = await config.convertToLlm(messages);

  // 第三步：构建 LLM Context，发请求
  const llmContext: Context = {
    systemPrompt: context.systemPrompt,
    messages: llmMessages,
    tools: context.tools,
  };
  const response = await streamFunction(config.model, llmContext, { ...config, signal });
  // ...
}
```

两步转换，各有分工：

- **transformContext**（可选）：在 AgentMessage 层面操作。比如消息太多时裁剪旧消息、注入外部上下文。这一步不需要知道 LLM 的消息格式。
- **convertToLlm**（必须）：把 AgentMessage 转成 LLM 认识的格式。自定义消息类型在这里被过滤或转换。

为什么不直接用 `Message[]` 跑全程？因为应用层经常需要在对话历史里塞一些 LLM 不需要看到的东西。

举个例子：用户切换了 Git 分支，你想在 UI 里显示一条"已切换到 feature-x 分支"的通知。这条消息需要出现在对话历史里（UI 要渲染），但不应该发给 LLM（它不需要知道这个）。用 `CustomAgentMessages` 扩展一个 `notification` 类型，`convertToLlm` 里过滤掉，就搞定了。

如果全程只用 `Message[]`，这种需求要么用 hack 塞进 system prompt，要么在 UI 层自己维护一个平行的消息列表。都很丑。

## Steering 和 FollowUp：运行中的两条队列

Agent 在执行工具时可能要跑好几秒甚至几十秒。这期间用户不是干等着——他可能想改需求、补充信息、或者排队下一个问题。

pi-mono 用两条队列解决这个问题：

**Steering（转向）**——"嘿，改个方向"

```typescript
// pi-agent-core/src/agent.ts
steer(m: AgentMessage) {
  this.steeringQueue.push(m);
}
```

Steering 消息的检查时机在每个工具执行完之后：

```typescript
// pi-agent-core/src/agent-loop.ts — executeToolCalls 内部
for (let index = 0; index < toolCalls.length; index++) {
  // ...执行当前工具...

  // 检查 steering
  if (getSteeringMessages) {
    const steering = await getSteeringMessages();
    if (steering.length > 0) {
      steeringMessages = steering;
      // 跳过剩余工具调用
      const remainingCalls = toolCalls.slice(index + 1);
      for (const skipped of remainingCalls) {
        results.push(skipToolCall(skipped, stream));
      }
      break;
    }
  }
}
```

关键行为：收到 steering 消息后，**剩余的工具调用直接跳过**。被跳过的工具会收到一个 `"Skipped due to queued user message."` 的错误结果。这意味着 steering 是一种中断——它打断当前执行流，让 LLM 重新评估。

**FollowUp（排队）**——"你忙完了告诉我，我还有事"

```typescript
// pi-agent-core/src/agent.ts
followUp(m: AgentMessage) {
  this.followUpQueue.push(m);
}
```

FollowUp 消息只在 Agent 准备结束时才被检查——内层循环退出后、真正 break 前。如果有排队消息，Agent 不会停，而是把消息设为 `pendingMessages`，回到外层循环继续处理。

这两种队列的优先级很清晰：
1. **Steering 优先级最高**：工具执行间隙就检查，有就立即中断
2. **FollowUp 最低**：只有 Agent 完全空闲时才消费

Agent 类还支持两种消费模式：`"all"`（一次性取出所有排队消息）和 `"one-at-a-time"`（每次只取一条）。默认是 `one-at-a-time`，更保守，避免一次塞太多上下文。

## 设计点评

说完机制，聊聊这个设计的 trade-off。

### 简洁：200 行搞定核心循环

`agent-loop.ts` 全文 417 行，其中 `runLoop` 不到 100 行。对比很多 Agent 框架动辄几千行的调度器，pi-mono 的选择是：**不做通用调度框架，只做一个够用的循环。**

没有任务图、没有并行编排、没有优先级队列。就是一个 while 循环，顺序执行。这意味着它不适合需要并行工具执行的场景——如果 LLM 一次返回 3 个工具调用，pi-mono 是串行执行的（`for` 循环遍历 `toolCalls`）。对 coding agent 来说这通常够了，但如果你想并行跑多个 API 请求，就得自己在工具内部处理。

### EventStream + generator：流式事件的好选择

整个循环通过 `EventStream` 发射事件，外部通过 `for await...of` 消费。这比回调模式干净得多——消费方可以随时 break，也天然支持背压。

```typescript
// pi-agent-core/src/agent-loop.ts
function createAgentStream(): EventStream<AgentEvent, AgentMessage[]> {
  return new EventStream<AgentEvent, AgentMessage[]>(
    (event: AgentEvent) => event.type === "agent_end",
    (event: AgentEvent) => (event.type === "agent_end" ? event.messages : []),
  );
}
```

`EventStream` 用两个 lambda 配置：第一个判断流是否结束，第二个从结束事件里提取最终结果。Agent 类那边消费时，就是标准的 async iteration：

```typescript
// pi-agent-core/src/agent.ts — _runLoop 内部
for await (const event of stream) {
  switch (event.type) {
    case "message_start": // ...
    case "message_update": // ...
    case "tool_execution_start": // ...
    // ...
  }
}
```

UI 层、日志层、测试层都可以各自订阅同一个事件流，互不干扰。

### 隐式状态机：简洁的代价

前面提到，stopReason 的处理没有用显式状态机。好处是代码短；代价是可测试性和可扩展性。

如果未来要加一个新的 stopReason（比如 `maxTokensReached`），你得在循环里找到所有相关的 if 分支，确保新状态不会被遗漏。显式状态机的好处是：每个状态的转换都是一个 case，漏了会被类型系统抓住。

但对 pi-mono 目前的规模来说，四种 stopReason 用隐式处理完全可控。这是"够用就好"的工程判断。

### 串行工具执行：有意的取舍

LLM 可以一次返回多个 tool_use。pi-mono 选择串行执行，而且在每个工具之间检查 steering 消息。

串行的好处是简单——不用处理并发竞争、不用想工具间的依赖关系、steering 中断的语义也更清晰（执行到哪个就停在哪个）。坏处是慢，如果 LLM 同时请求读 5 个文件，本可以并行完成的事要串行等。

这个取舍对 coding agent 来说是合理的。大部分工具调用（读文件、写文件、跑命令）有隐式的顺序依赖，并行执行反而可能出问题。但如果要把这个引擎用于网络爬虫或数据采集类 Agent，串行就会成为明显瓶颈。

### convertToLlm 的调用时机

每一轮 LLM 调用都会重新执行 `convertToLlm`，把**整个** AgentMessage 历史转一遍。如果对话很长、转换逻辑复杂，这会成为性能热点。

但反过来，这也意味着 `transformContext`（上下文裁剪）可以在每轮都动态调整——比如前 10 轮保留全部上下文，之后开始压缩。这种灵活性在长对话场景下很有价值。

## 思考题

Agent Loop 目前是顺序执行工具调用的。假设你要支持并行执行（LLM 返回 3 个 tool_use，同时跑），需要改 `runLoop` 的哪些部分？Steering 的中断语义又该怎么调整——是等所有并行工具都完成再检查，还是任何一个完成就检查？两种选择各有什么后果？
