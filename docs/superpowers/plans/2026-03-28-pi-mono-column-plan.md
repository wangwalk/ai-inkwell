# 深入 AI Agent — pi-mono 源码拆解：实施计划

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 完成 6 篇源码拆解文章，从全景到细节逐层解析 pi-mono 的 Agent 应用架构。

**Architecture:** 每篇文章是一个独立 Task，按发布顺序执行。每个 Task 包含：源码研读 → 大纲 → 写作 → 自审 → 提交。文章存放在 `01-Drafts/深入AI-Agent-pi-mono源码拆解/`。

**Tech Stack:** Obsidian Markdown，pi-mono 源码位于 `~/Documents/wangwalk/pi-mono/`

**写作规范：**
- 遵循 CLAUDE.md 写作风格（朋友式对话、干货优先、短段落、有观点）
- 遵循专栏模板的四段结构：场景切入 → 架构拆解 → 设计点评 → 思考题
- 源码摘录注明包名和文件路径，可用 `// ...` 省略无关部分
- 篇幅 3000-5000 字
- 中文写作，技术术语保留英文（如 Agent Loop、stopReason、Context）

---

## Task 0: 创建专栏目录

**Files:**
- Create: `01-Drafts/深入AI-Agent-pi-mono源码拆解/`（目录）

- [ ] **Step 1: 创建目录**

```bash
mkdir -p "01-Drafts/深入AI-Agent-pi-mono源码拆解"
```

- [ ] **Step 2: 提交**

```bash
git add "01-Drafts/深入AI-Agent-pi-mono源码拆解" && git commit -m "chore: create draft directory for pi-mono column"
```

---

## Task 1: 第 1 篇 — 全景：一个 Agent 应用长什么样

**场景切入**：你在终端里输入一句话，Agent 读了文件、改了代码、跑了测试——背后经过了哪些模块？

**Files:**
- Create: `01-Drafts/深入AI-Agent-pi-mono源码拆解/01-全景-一个Agent应用长什么样.md`

**需要研读的源码（按阅读顺序）：**
- `package.json`（根目录）— 工作区结构和包依赖
- `packages/ai/src/index.ts` — pi-ai 对外暴露什么
- `packages/agent/src/index.ts` — pi-agent-core 对外暴露什么
- `packages/coding-agent/src/index.ts` + `src/core/index.ts` — 产品层入口
- `packages/coding-agent/src/cli.ts` — 用户输入从哪里进来

- [ ] **Step 1: 研读源码**

阅读上述文件，重点搞清楚：
1. 7 个包各自的职责边界
2. 包之间的依赖关系（谁依赖谁）
3. 一条用户输入从 cli.ts 进入后，经过哪些模块到达 LLM，再回来

记录关键发现，标注要摘录的源码片段和文件路径。

- [ ] **Step 2: 写大纲**

基于模板四段结构列出要点：
1. 场景切入：终端输入一句话，Agent 干了一堆事
2. 架构拆解：7 包总览表 → 三层依赖链图 → 请求路径走读
3. 设计点评：为什么分三层不一个大包（关注点分离、可复用性、独立演进）
4. 思考题

- [ ] **Step 3: 从模板创建文章文件**

从 `templates/深入AI-Agent源码拆解.md` 创建，填入 frontmatter：
- title: "全景：一个 Agent 应用长什么样"
- date: 当天日期
- tags: 加上 `monorepo`, `overview`

- [ ] **Step 4: 写正文**

按大纲展开，注意：
- 7 包总览用表格呈现（包名 | 一句话职责 | 核心/应用）
- 依赖链用 ASCII 图或文字描述，不依赖外部渲染
- 请求路径走读点到为止，每个模块一两句话，留悬念给后续篇
- 点评部分要有自己的判断，不只是描述

- [ ] **Step 5: 自审**

检查清单：
- [ ] 开头是否直接切入，没有寒暄铺垫？
- [ ] 段落是否短（适配手机阅读）？
- [ ] 源码摘录是否标注了文件路径？
- [ ] 设计点评是否有自己的观点（不只是转述）？
- [ ] 结尾是否留了思考题而非总结复述？
- [ ] 字数在 3000-5000 范围内？

- [ ] **Step 6: 提交**

```bash
git add "01-Drafts/深入AI-Agent-pi-mono源码拆解/01-全景-一个Agent应用长什么样.md"
git commit -m "draft: 第1篇 全景——一个Agent应用长什么样"
```

---

## Task 2: 第 2 篇 — Agent 引擎：循环的心跳

**场景切入**：用户问一次，LLM API 可能被调了 5 次——Agent 和 Chatbot 的本质区别在这。

**Files:**
- Create: `01-Drafts/深入AI-Agent-pi-mono源码拆解/02-Agent引擎-循环的心跳.md`

**需要研读的源码：**
- `packages/agent/src/agent-loop.ts` — agentLoop 核心循环实现
- `packages/agent/src/types.ts` — AgentMessage、AgentEvent、stopReason、steering/followup 类型
- `packages/agent/src/agent.ts` — Agent 类封装和状态管理

- [ ] **Step 1: 研读源码**

重点搞清楚：
1. agentLoop 的 while 循环结构：什么条件进入、什么条件退出
2. stopReason 的几种值（endTurn / toolUse / error / aborted）如何驱动流程
3. AgentMessage 和 LLM Message 的区别：AgentMessage 有哪些额外类型
4. convertToLlm 怎么过滤消息
5. steering 和 followUp 两个队列的触发时机和处理方式
6. 工具执行是串行还是并行

标注关键代码段落。

- [ ] **Step 2: 写大纲**

1. 场景切入：一次提问触发 N 次 LLM 调用——Agent ≠ Chatbot
2. 架构拆解：
   - agentLoop 的循环结构（画出流程图用 ASCII）
   - stopReason 状态机
   - 两层消息：AgentMessage → convertToLlm → Message
   - Steering（中途纠正）vs FollowUp（排队等候）
3. 设计点评：循环的简洁性、generator 的使用、可能的瓶颈
4. 思考题

- [ ] **Step 3: 从模板创建文章文件**

填入 frontmatter：
- title: "Agent 引擎：循环的心跳"
- tags: 加上 `agent-loop`, `tool-calling`

- [ ] **Step 4: 写正文**

注意：
- agentLoop 的核心循环用简化版源码展示，标注 `packages/agent/src/agent-loop.ts`
- stopReason 用表格对比各值和对应行为
- 两层消息用流程图：`AgentMessage[] → transformContext() → convertToLlm() → Message[]`
- Steering/FollowUp 用具体场景说明（比如：用户在工具执行期间发了新指令）
- 这是全系列最核心的一篇，点评部分要够深

- [ ] **Step 5: 自审**

同 Task 1 自审检查清单。

- [ ] **Step 6: 提交**

```bash
git add "01-Drafts/深入AI-Agent-pi-mono源码拆解/02-Agent引擎-循环的心跳.md"
git commit -m "draft: 第2篇 Agent引擎——循环的心跳"
```

---

## Task 3: 第 3 篇 — LLM 抽象层：统一 20+ 提供商的代价

**场景切入**：把 OpenAI 换成 Claude，代码一行不改——这层抽象是怎么做到的？

**Files:**
- Create: `01-Drafts/深入AI-Agent-pi-mono源码拆解/03-LLM抽象层-统一20家提供商的代价.md`

**需要研读的源码：**
- `packages/ai/src/types.ts` — Context、Model、Api/Provider 类型定义
- `packages/ai/src/stream.ts` — streamSimple 函数和 provider 路由
- `packages/ai/src/models.ts` — Model 注册表和查找逻辑
- `packages/ai/src/api-registry.ts` — ApiProvider 注册系统
- `packages/ai/src/utils/event-stream.ts` — 统一事件流
- `packages/ai/src/providers/register-builtins.ts` — 内置 provider 注册
- 任选 1-2 个具体 provider（如 `providers/anthropic.ts`、`providers/openai-responses.ts`）看实现细节

- [ ] **Step 1: 研读源码**

重点搞清楚：
1. streamSimple 怎么根据 model 选择 provider
2. 每个 provider 需要实现什么（stream 函数 + 消息转换器）
3. 统一事件流有哪些事件类型
4. Context 的数据结构：包含什么、怎么序列化
5. Model 注册表的设计：类型安全怎么实现

- [ ] **Step 2: 写大纲**

1. 场景切入：换 provider 不改代码
2. 架构拆解：
   - Provider 插件架构图
   - streamSimple 的路由机制
   - 统一事件流（text_delta / toolcall_start / done）
   - Context 和 Model 的类型设计
3. 设计点评：统一抽象的收益 vs "最小公约数"问题（有些 provider 的独特能力被抹平了吗？）
4. 思考题

- [ ] **Step 3: 从模板创建文章文件**

- title: "LLM 抽象层：统一 20+ 提供商的代价"
- tags: 加上 `llm`, `provider`, `abstraction`

- [ ] **Step 4: 写正文**

注意：
- 对比两个 provider 的原始 API 差异，再展示统一后的样子——让读者感受抽象的价值
- "最小公约数"点评是这篇的亮点：不只说好，也要说抽象的代价

- [ ] **Step 5: 自审**

同 Task 1 自审检查清单。

- [ ] **Step 6: 提交**

```bash
git add "01-Drafts/深入AI-Agent-pi-mono源码拆解/03-LLM抽象层-统一20家提供商的代价.md"
git commit -m "draft: 第3篇 LLM抽象层——统一20家提供商的代价"
```

---

## Task 4: 第 4 篇 — 产品层：生产环境的脏活

**场景切入**：Agent 引擎只管循环，但对话断了要恢复、context 满了要压缩、API 限流要重试——谁来干？

**Files:**
- Create: `01-Drafts/深入AI-Agent-pi-mono源码拆解/04-产品层-生产环境的脏活.md`

**需要研读的源码：**
- `packages/coding-agent/src/core/agent-session.ts` — AgentSession 生命周期和持久化
- `packages/coding-agent/src/core/compaction/compaction.ts` — 压缩核心逻辑
- `packages/coding-agent/src/core/compaction/index.ts` — 压缩工具函数
- `packages/coding-agent/src/core/session-manager.ts` — 会话头、条目类型、持久化接口
- `packages/coding-agent/src/core/system-prompt.ts` — System Prompt 动态拼装
- `packages/coding-agent/src/core/messages.ts` — 自定义消息类型

- [ ] **Step 1: 研读源码**

重点搞清楚：
1. AgentSession 在 Agent 之上包了什么（重试、压缩、持久化、事件转发）
2. 压缩的触发条件、切割点选择、摘要生成策略、增量更新
3. JSONL 会话存储格式和树形分支机制
4. System Prompt 的拼装顺序和各部分来源
5. 自定义消息类型有哪些（BashExecution、CompactionSummary 等）

- [ ] **Step 2: 写大纲**

1. 场景切入：引擎之外的脏活
2. 架构拆解：
   - AgentSession vs Agent 职责对比表
   - 压缩策略（触发 → 切割 → 摘要 → 替换）
   - 会话持久化和分支
   - System Prompt 拼装流水线
3. 设计点评：关注点分离的好处 + 这层复杂度是否值得
4. 思考题

- [ ] **Step 3: 从模板创建文章文件**

- title: "产品层：生产环境的脏活"
- tags: 加上 `session`, `compaction`, `persistence`

- [ ] **Step 4: 写正文**

注意：
- AgentSession vs Agent 用对比表格直观展示
- 压缩策略是重点，用具体数字说明（如 16384 token 预留）
- System Prompt 拼装用编号列表展示顺序

- [ ] **Step 5: 自审**

同 Task 1 自审检查清单。

- [ ] **Step 6: 提交**

```bash
git add "01-Drafts/深入AI-Agent-pi-mono源码拆解/04-产品层-生产环境的脏活.md"
git commit -m "draft: 第4篇 产品层——生产环境的脏活"
```

---

## Task 5: 第 5 篇 — 终端 UI：无闪烁的差量渲染

**场景切入**：Agent 在流式输出 Markdown、同时底部有个 spinner 在转——终端怎么做到不闪不乱？

**Files:**
- Create: `01-Drafts/深入AI-Agent-pi-mono源码拆解/05-终端UI-无闪烁的差量渲染.md`

**需要研读的源码：**
- `packages/tui/src/tui.ts` — TUI 类，差量渲染核心逻辑
- `packages/tui/src/terminal.ts` — Terminal 接口和 ProcessTerminal 实现
- `packages/tui/src/components/editor.ts` — Editor 组件
- `packages/tui/src/components/markdown.ts` — Markdown 渲染组件
- `packages/tui/src/index.ts` — 组件体系总览

- [ ] **Step 1: 研读源码**

重点搞清楚：
1. TUI 的渲染策略：首次渲染 vs 普通更新 vs 宽度变化
2. CSI 2026 同步输出怎么用的
3. 差量对比的粒度（行级？字符级？）
4. 组件体系：Container、Editor、Markdown、Overlay 的关系
5. CJK 输入的 CURSOR_MARKER 机制

- [ ] **Step 2: 写大纲**

1. 场景切入：流式 Markdown + spinner 不闪不乱
2. 架构拆解：
   - 三种渲染路径（首次 / 更新 / 全量重绘）
   - CSI 2026 同步输出协议
   - 组件树和生命周期
   - CJK IME 处理
3. 设计点评：终端 UI 的取舍——为什么不用 Ink/Blessed 等现成方案
4. 思考题

- [ ] **Step 3: 从模板创建文章文件**

- title: "终端 UI：无闪烁的差量渲染"
- tags: 加上 `tui`, `rendering`, `terminal`

- [ ] **Step 4: 写正文**

注意：
- 渲染策略用伪代码流程图解释，比纯文字更清晰
- CSI 2026 可以简单解释终端同步输出的原理
- 读者可能对 CJK IME 处理没概念，需要用具体例子说明

- [ ] **Step 5: 自审**

同 Task 1 自审检查清单。

- [ ] **Step 6: 提交**

```bash
git add "01-Drafts/深入AI-Agent-pi-mono源码拆解/05-终端UI-无闪烁的差量渲染.md"
git commit -m "draft: 第5篇 终端UI——无闪烁的差量渲染"
```

---

## Task 6: 第 6 篇 — 插件系统：不改源码扩展一切

**场景切入**：想给 Agent 加一个"搜索网页"的能力，不 fork 源码能做到吗？

**Files:**
- Create: `01-Drafts/深入AI-Agent-pi-mono源码拆解/06-插件系统-不改源码扩展一切.md`

**需要研读的源码：**
- `packages/coding-agent/src/core/extensions/types.ts` — Extension 接口、生命周期钩子定义
- `packages/coding-agent/src/core/extensions/runner.ts` — ExtensionRunner 钩子执行和快捷键管理
- `packages/coding-agent/src/core/extensions/loader.ts` — 扩展发现和动态加载（jiti）
- `packages/coding-agent/src/core/extensions/wrapper.ts` — 工具包装和扩展上下文
- `packages/coding-agent/src/core/extensions/index.ts` — 扩展系统导出

- [ ] **Step 1: 研读源码**

重点搞清楚：
1. ExtensionAPI 提供了哪些注册能力（工具、命令、快捷键、事件监听、UI）
2. 生命周期钩子有哪些、执行时机
3. 扩展怎么被发现和加载的（本地目录 / pi 包 / npm）
4. 权限门控和路径保护怎么实现
5. 扩展和内置工具的边界在哪

- [ ] **Step 2: 写大纲**

1. 场景切入：加一个搜索网页能力，不 fork 源码
2. 架构拆解：
   - ExtensionAPI 能力清单
   - 生命周期钩子和事件流
   - 加载机制（发现 → 加载 → 注册 → 运行）
   - 权限和安全
3. 设计点评：什么该内置什么该插件化？插件系统的边界问题
4. 思考题

- [ ] **Step 3: 从模板创建文章文件**

- title: "插件系统：不改源码扩展一切"
- tags: 加上 `extension`, `plugin`, `hooks`

- [ ] **Step 4: 写正文**

注意：
- 用一个具体的扩展示例贯穿全篇（比如注册一个自定义工具）
- 钩子列表用表格呈现（钩子名 | 触发时机 | 能做什么）
- 点评的重点：插件化的边界判断——开放太多会乱，开放太少没用

- [ ] **Step 5: 自审**

同 Task 1 自审检查清单。

- [ ] **Step 6: 提交**

```bash
git add "01-Drafts/深入AI-Agent-pi-mono源码拆解/06-插件系统-不改源码扩展一切.md"
git commit -m "draft: 第6篇 插件系统——不改源码扩展一切"
```
