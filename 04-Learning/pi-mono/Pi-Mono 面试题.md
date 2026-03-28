# Pi-Mono 项目面试题

> 基于对 pi-mono 源码的学习整理，涵盖架构理解、设计决策和场景应用。

---

## 一、基础题 — 架构理解

### Q1: pi-mono 的三层架构是什么？各自职责？

**参考回答：**

| 层 | 包名 | 职责 |
|---|---|---|
| LLM 抽象层 | `pi-ai` | 统一 20+ LLM 提供商的接口，屏蔽各家 API 差异 |
| Agent 引擎 | `pi-agent-core` | 自动化 Agent 循环（调 LLM → 执行工具 → 再调 LLM） |
| 产品层 | `pi-coding-agent` | 完整的终端编码助手，包含工具、会话管理、UI、插件系统 |

依赖链：`pi-ai` ← `pi-agent-core` ← `pi-coding-agent`

---

### Q2: Tool Calling 机制是怎么工作的？模型真的会执行工具吗？

**参考回答：**

不会。模型**只是发出调用请求**，不执行任何东西。流程：

1. 你在 Context 里声明有哪些工具（名字、描述、参数 schema）
2. 模型看到后自己决定要不要调用，返回 `{ name: "get_time", arguments: {...} }`
3. **你的代码**决定是否执行、怎么执行、返回什么给模型

这么设计是因为**模型不可信** — 由你的代码控制安全、权限、沙箱。

---

### Q3: 用户问一次，LLM API 被调用几次？

**参考回答：**

可能是多次。Agent 循环的逻辑：

```
用户提问 → 第1次调 LLM
  → stopReason = tool_use → 执行工具 → 第2次调 LLM
    → stopReason = tool_use → 执行工具 → 第3次调 LLM
      → stopReason = end_turn → 结束
```

调用次数取决于模型调了几次工具。用户感知到一次对话，实际可能是 N 次 LLM API 调用。这就是 **Agent 和 Chatbot 的核心区别**。

---

### Q4: stream 返回的事件流有哪些关键事件？它们解决什么问题？

**参考回答：**

关键事件：
- `text_delta` — 文本增量（实现打字机效果）
- `toolcall_start` / `toolcall_end` — 工具调用请求的开始和完成
- `done` — 响应结束

这些事件的价值在于**统一**。OpenAI、Anthropic、Gemini 的原始流式格式各不相同，`pi-ai` 把它们翻译成统一事件，上层代码只写一套逻辑。

---

### Q5: stopReason 有哪几种，分别表示什么？

**参考回答：**

| stopReason | 含义 | Agent 循环的行为 |
|---|---|---|
| `end_turn` | 模型说完了 | 正常退出循环 |
| `tool_use` | 模型想调用工具 | 执行工具，继续循环 |
| `error` | API 调用出错 | 退出循环（AgentSession 层可能重试） |
| `aborted` | 用户取消（Ctrl+C） | 退出循环 |

---

## 二、深入题 — 设计决策

### Q6: 为什么 Agent 有两层消息（AgentMessage vs LLM Message）？

**参考回答：**

因为应用层需要**比 LLM 能理解的更丰富的消息类型**。

- `AgentMessage` — 可以包含 UI 通知、状态消息、bash 执行记录、自定义消息等
- LLM 只认三种：`user`、`assistant`、`toolResult`

转换管道：

```
AgentMessage[] → transformContext() → AgentMessage[] → convertToLlm() → Message[]
```

- `transformContext`：Agent 消息层面的操作（裁剪旧消息、注入上下文）
- `convertToLlm`：过滤掉 LLM 不理解的消息，转成标准格式

这样 Agent 内部可以自由扩展消息类型，不受 LLM 格式限制。

---

### Q7: Steering Messages 和 FollowUp Messages 的区别是什么？

**参考回答：**

两者都是用户在 Agent 运行期间发的消息，但行为不同：

| 类型 | 时机 | 用途 |
|---|---|---|
| Steering | 工具执行后、下次 LLM 调用前插入 | 中途纠正 Agent 方向 |
| FollowUp | Agent 完全结束后才发送 | 排队等 Agent 做完再说 |

设计原因：工具执行可能很慢（比如跑 30 秒的 shell 命令），这期间用户可能想打断或追加指令。两种队列给用户不同粒度的控制。

---

### Q8: Context 压缩（Compaction）的具体策略是什么？

**参考回答：**

**触发条件**：`contextTokens > contextWindow - reserveTokens(16384)`

**步骤**：
1. **找切割点** — 从最新消息往回数约 20000 token，在 user/assistant 消息处切（永远不在 toolResult 处切，因为 toolResult 必须跟着 toolCall）
2. **调 LLM 生成结构化摘要** — 按固定格式（Goal / Progress / Key Decisions / Next Steps）总结被丢弃的消息
3. **替换历史** — 丢弃旧消息，插入摘要消息，保留最近的消息继续对话

**增量更新**：如果之前已经压缩过，不是从头总结，而是把新消息的进展合并进已有摘要。这样多次压缩不会丢失历史信息。

**Split Turn**：如果切割点落在一个 turn 中间，会为前半段额外生成一个 turn prefix 摘要，两个摘要并行生成后合并。

---

### Q9: AgentSession 相比 Agent（引擎层）多做了什么？

**参考回答：**

`Agent` 是纯循环引擎，只管"调模型 → 执行工具 → 循环"。

`AgentSession` 包装了生产环境需要的所有"脏活"：

| 能力 | 说明 |
|---|---|
| 会话持久化 | 每条消息实时写入磁盘，重启可恢复 |
| 自动重试 | API 限流/过载时自动等待重试 |
| 自动压缩 | context window 快满时自动触发压缩 |
| 插件事件转发 | 把 Agent 事件翻译后分发给插件 |
| 消息队列管理 | 维护 steering / followUp 两个队列 |

设计思想是**关注点分离**：引擎层保持纯粹，产品层的复杂性在外面包。

---

### Q10: System Prompt 是怎么动态构建的？

**参考回答：**

按固定顺序拼接：

```
① 身份定义 + 可用工具列表
② Guidelines（根据激活了哪些工具动态生成）
③ Pi 文档路径
④ appendSystemPrompt（插件追加的内容）
⑤ Context Files（CLAUDE.md 等项目配置）
⑥ Skills（可用技能的名字和描述 — 延迟加载，不塞内容）
⑦ 当前日期时间 + 工作目录
```

亮点：
- **Guidelines 和工具联动** — 有 grep 工具时告诉模型"优先用 grep"，没有时才说"用 bash"
- **Skills 是延迟加载** — 只列名字，模型需要时自己 read 文件，省 token
- **Context Files 就是 CLAUDE.md** — 每次对话都注入，不是"记忆"

---

## 三、场景题 — 实际应用

### Q11: 如果要给 pi 增加一个"搜索网页"的能力，你会怎么做？

**参考回答：**

写一个 Extension，注册一个新工具：

```typescript
ctx.registerTool({
    name: "search_web",
    description: "Search the internet for information",
    parameters: Type.Object({ query: Type.String() }),
    async execute(id, params, signal) {
        const results = await fetchSearchAPI(params.query);
        return { content: formatResults(results) };
    }
});
```

不需要改 pi 源码。工具会自动出现在模型的可用工具列表里，模型会在需要时自主调用它。

---

### Q12: 用户说"对话变慢了，每轮要等很久"，从架构角度可能是什么原因？

**参考回答：**

按架构层次分析：

1. **Context 太长**（pi-ai 层）— 消息历史太多，每次 LLM 调用传的 token 很多。解决：触发 compaction 压缩
2. **工具执行慢**（agent-core 层）— 某个工具（比如 bash 执行长命令）耗时太久。与 LLM 调用无关
3. **API 限流/重试**（AgentSession 层）— API 返回 429/overloaded，AgentSession 在自动重试等待
4. **每轮工具调用太多**（agent-core 层）— 模型一轮里调了很多工具，每个都要等执行结果再进下一轮

---

### Q13: 如果你想在每次 Agent 调用 bash 工具前加一个权限确认，怎么实现？

**参考回答：**

在 Extension 中监听 `tool_call` 事件：

```typescript
on("tool_call", async (event, ctx) => {
    if (event.toolName === "bash") {
        const confirmed = await ctx.ui.confirm(
            "执行确认",
            `即将执行: ${event.args.command}`
        );
        if (!confirmed) {
            return { abort: true };  // 拦截这次工具调用
        }
    }
});
```

利用插件系统的 Hook 机制，在工具执行前拦截，不需要改内置工具的代码。

---

### Q14: pi-ai 的 Context 是"可序列化、可跨 provider 传递"的，这在实际中有什么用？

**参考回答：**

实际用途：

1. **模型切换** — 用 GPT-4 对话到一半，切换到 Claude 继续，因为 Context 格式与 provider 无关
2. **会话持久化** — Context 是纯数据，可以直接 JSON 序列化存盘，下次加载恢复
3. **Agent 间协作** — 一个 Agent 的对话上下文可以传递给另一个 Agent 继续处理
4. **调试** — 把 Context dump 出来分析模型的决策依据

---

## 四、架构全景图

```
用户输入
  ↓
[Extension: input]              ← 预处理输入
  ↓
[Extension: before_agent_start] ← 修改 systemPrompt
  ↓
AgentSession                    ← 重试/压缩/持久化
  ↓
Agent Loop (pi-agent-core)      ← while(true) 循环
  │
  ├─ [Extension: context]       ← 修改消息列表
  ├─ convertToLlm()             ← AgentMessage → Message
  ├─ stream/complete (pi-ai)    ← 统一调 LLM API
  ├─ [Extension: tool_call]     ← 拦截工具调用
  ├─ 执行工具
  ├─ [Extension: tool_result]   ← 修改工具结果
  └─ 结果塞回 context → 继续循环
  ↓
[Extension: agent_end]          ← 对话结束
  ↓
响应给用户
```
