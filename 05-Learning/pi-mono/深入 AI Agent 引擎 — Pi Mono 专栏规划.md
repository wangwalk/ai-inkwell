# 深入 AI Agent 引擎 — Pi Mono

> 专栏目录规划

---

| # | 标题 | 核心内容 |
|---|------|---------|
| 01 | 全景：一个 AI Agent 框架的分层架构 | 项目定位、7 个包的依赖关系、LLM 抽象→Agent 引擎→产品层三层总览 |
| 02 | LLM 抽象：如何用一套接口统一 20+ 模型提供商 | Provider 注册与懒加载、统一事件流、流式 partial JSON 解析、Token 计费与 Prompt Cache |
| 03 | Agent 循环：一次提问背后的 N 次 LLM 调用 | 循环控制、stopReason 驱动、工具执行与结果回注、Agent 与 Chatbot 的本质区别 |
| 04 | 消息系统：Agent 需要比 LLM 更丰富的对话模型 | 应用消息 vs LLM 消息的设计动机、上下文转换管道、自定义消息类型、Steering 与 FollowUp 队列 |
| 05 | 上下文压缩：让无限对话成为可能 | 触发条件、切割点选择、结构化摘要生成、增量更新与多轮压缩 |
| 06 | 会话管理：重试、压缩、持久化 | JSONL 树形存储、自动重试、自动压缩调度、事件转发、会话分支与 Fork |
| 07 | 终端渲染：如何做到流式输出不闪烁 | 三策略差分渲染、同步输出、ANSI 码保持、内联图片协议、CJK 输入法适配 |
| 08 | 插件体系：工具注册、事件钩子与 System Prompt 构建 | Extension 机制、Skills 延迟加载、System Prompt 动态拼接、包分发 |
| 09 | Slack Bot：一个能自我管理的 AI 助手 | Docker 沙箱、自建 CLI 工具、定时任务调度、频道隔离与记忆 |
| 10 | GPU 部署：用 vLLM 跑开源模型 | 自动化部署、多 GPU 分配、Tool Calling 配置、成本与性能权衡 |
