---
series: 深入AI-Agent-pi-mono源码拆解
title: "全景：一个 Agent 应用长什么样"
date: "2026-03-28"
status: draft
tags:
  - agent
  - architecture
  - source-code
  - monorepo
  - overview
published:
  公众号:
  知乎:
  博客:
---

你在终端里敲了一句"把这个函数的错误处理补上"，回车。几秒后，Agent 读了源文件、改了代码、跑了测试，告诉你搞定了。

这背后到底经过了哪些模块？

pi-mono 是一个开源的 coding agent 框架，结构清晰，拆包讲究。今天我们不深入任何一层，而是先用一张全景图，搞清楚整个应用的骨架。

## 7 个包，各司其职

pi-mono 是个 monorepo，`packages/` 下有 7 个包。但不是每个都跟 Agent 核心逻辑相关——有些是基础设施，有些是独立产品。

| 包名 | npm 名 | 一句话职责 | 角色 |
|------|--------|-----------|------|
| `ai` | `@mariozechner/pi-ai` | 统一 LLM API：封装 Anthropic、OpenAI、Google 等 20+ 家提供商 | 核心 |
| `agent` | `@mariozechner/pi-agent-core` | Agent Loop 引擎：消息循环、工具调用、状态管理 | 核心 |
| `coding-agent` | `@mariozechner/pi-coding-agent` | 产品层：CLI 入口、Session 管理、内置工具、插件系统 | 核心（应用） |
| `tui` | `@mariozechner/pi-tui` | 终端 UI 库：差量渲染，不闪烁 | 基础设施 |
| `web-ui` | `@mariozechner/pi-web-ui` | Web 聊天组件 | 应用 |
| `mom` | `@mariozechner/pi-mom` | Slack 机器人，把消息委托给 coding agent | 应用 |
| `pods` | `@mariozechner/pi` | GPU Pod 管理工具（vLLM 部署） | 应用 |

核心链路只涉及前三个包。`tui` 是渲染层，`web-ui`/`mom`/`pods` 是独立产品，可以完全忽略。

## 三层依赖链

核心的三层关系非常干净：

```
pi-ai  ←  pi-agent-core  ←  pi-coding-agent
 LLM 抽象      Agent 引擎        产品层
```

依赖方向是单向的：

- **pi-ai** 不依赖任何同仓库的包。它只管一件事：给我一个 Model 和 Context，我帮你流式调 LLM。
- **pi-agent-core** 依赖 pi-ai。它在 LLM 能力之上，加了 Agent Loop——消息循环、工具调用、停止判断。
- **pi-coding-agent** 依赖前两者。它把 Agent 引擎包装成一个完整的命令行产品——CLI 参数解析、Session 持久化、内置的 read/edit/bash/write 工具、插件系统。

从 `package.json` 的依赖声明能直接验证：

```json
// pi-agent-core/package.json
"dependencies": {
  "@mariozechner/pi-ai": "^0.52.9"
}

// pi-coding-agent/package.json
"dependencies": {
  "@mariozechner/pi-agent-core": "^0.52.9",
  "@mariozechner/pi-ai": "^0.52.9",
  // ...
}
```

注意 `pi-coding-agent` 同时依赖了 pi-ai 和 pi-agent-core，而不是只依赖 pi-agent-core。这意味着产品层会直接使用 LLM 抽象层的类型和工具函数，不完全通过 Agent 引擎中转。

## 一条消息的完整旅程

你输入一句话，到底发生了什么？我们走马观花过一遍。

### 1. CLI 入口：cli.ts

```typescript
// pi-coding-agent/src/cli.ts
process.title = "pi";
import { main } from "./main.js";
main(process.argv.slice(2));
```

就这么三行。所有活都交给 `main()`。

### 2. 参数解析与初始化：main.ts

`main()` 做的事很多，但核心路径是：

```typescript
// pi-coding-agent/src/main.ts
const settingsManager = SettingsManager.create(cwd, agentDir);
const authStorage = new AuthStorage();
const modelRegistry = new ModelRegistry(authStorage, getModelsPath());
const resourceLoader = new DefaultResourceLoader({ cwd, agentDir, settingsManager, /* ... */ });
await resourceLoader.reload();
// ...
const { session } = await createAgentSession(sessionOptions);
// ...
const mode = new InteractiveMode(session, { /* ... */ });
await mode.run();
```

先创建一堆管理器（设置、认证、模型注册、资源加载），然后通过 `createAgentSession()` 拿到一个 `AgentSession`，最后根据模式（交互/打印/RPC）启动运行。

### 3. SDK 工厂：createAgentSession

```typescript
// pi-coding-agent/src/core/sdk.ts
import { Agent } from "@mariozechner/pi-agent-core";

// createAgentSession 内部：
// 1. 初始化 Agent（来自 pi-agent-core）
// 2. 注册内置工具（read, bash, edit, write, grep, find, ls）
// 3. 加载插件
// 4. 构建 AgentSession（产品层的包装）
```

这里是三层汇合的地方。`Agent` 来自 pi-agent-core，`Model` 来自 pi-ai，工具和 Session 管理是 pi-coding-agent 自己的。

### 4. Agent Loop：消息循环

当用户输入到达后，`AgentSession` 调用 pi-agent-core 的 `agentLoop()`：

```typescript
// pi-agent-core/src/agent-loop.ts
export function agentLoop(
  prompts: AgentMessage[],
  context: AgentContext,
  config: AgentLoopConfig,
  signal?: AbortSignal,
): EventStream<AgentEvent, AgentMessage[]> {
  // ...
  stream.push({ type: "agent_start" });
  stream.push({ type: "turn_start" });
  await runLoop(currentContext, newMessages, config, signal, stream);
}
```

`runLoop` 是核心循环：发消息给 LLM → 拿到回复 → 如果回复里有工具调用就执行 → 把结果放回上下文 → 再发给 LLM → 直到 LLM 说"我完事了"（stopReason 不是 tool_use）。

### 5. LLM 调用：pi-ai 的 streamSimple

最底层，Agent Loop 通过 pi-ai 的 `streamSimple()` 发请求：

```typescript
// pi-ai/src/stream.ts
export function streamSimple<TApi extends Api>(
  model: Model<TApi>,
  context: Context,
  options?: SimpleStreamOptions,
): AssistantMessageEventStream {
  const provider = resolveApiProvider(model.api);
  return provider.streamSimple(model, context, options);
}
```

`resolveApiProvider` 根据 model 的 API 类型找到对应的 provider 实现（Anthropic、OpenAI、Google...），然后调它的流式接口。上层完全不需要知道底下是哪家的 API。

### 6. 工具执行

LLM 返回的消息里如果包含 tool_use，Agent Loop 会找到对应的工具定义并执行。这些工具（read、edit、bash、write 等）定义在 pi-coding-agent 里，但执行调度由 pi-agent-core 的 Loop 控制。

### 7. 回到用户

工具执行完，结果塞回消息上下文，再送一轮给 LLM。LLM 决定继续调工具还是直接回复用户。最终文本通过 `InteractiveMode` 渲染到终端。

整个流程：**CLI → main → createAgentSession → AgentSession → Agent Loop → streamSimple → Provider → LLM**，返回后再经工具执行循环，直到结束。

## 为什么要分三层？

一个大包不行吗？行，但会付出代价。

**pi-ai 独立的好处最明显。** LLM 提供商的 API 格式千差万别，version 迭代频繁。把这层独立出来，意味着加一个新 provider 不用碰 Agent 逻辑。它甚至有自己的 CLI（`pi-ai`），可以单独用来测试模型。

**pi-agent-core 独立的好处是复用。** Agent Loop 的核心逻辑——消息循环、工具调度、上下文管理——跟"coding"没关系。理论上你可以用同一套引擎做一个客服 Agent、一个数据分析 Agent。实际上 `mom`（Slack bot）就是另一个基于同一引擎的产品。

**pi-coding-agent 是"脏活"集中地。** CLI 参数解析、Session 持久化、文件工具的具体实现、插件加载——这些都是产品级的需求，跟引擎的抽象程度不在一个层面。如果把这些混进 agent-core，那个包的 API 表面积会爆炸。

当然，分层也有成本。从 `pi-coding-agent/src/index.ts` 的 300+ 行导出可以看出，产品层需要重新导出大量底层类型给外部消费者。包间的版本同步也需要额外脚本（`scripts/sync-versions.js`）。

但总体来说，这个分层是值得的。我见过太多 Agent 框架把所有东西塞进一个包，结果改个 provider 要跑全量测试，加个工具要理解整个 LLM 调用链。pi-mono 的分层让每一层的心智负担都可控。

## 思考题

pi-agent-core 的 `Agent` 类接收一个 `convertToLlm` 函数，把 `AgentMessage[]` 转成 LLM 能理解的 `Message[]`。这意味着 Agent 内部的消息格式和 LLM 的消息格式是解耦的。

问题是：这种解耦在什么场景下会真正派上用场？如果你要在 Agent 的对话历史里插入一条"系统通知"（比如"用户切换了分支"），它不应该发给 LLM 但需要在 UI 里显示——这条消息应该放在哪一层处理？
