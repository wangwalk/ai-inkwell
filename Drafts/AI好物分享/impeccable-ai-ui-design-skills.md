---
series: AI好物分享
title: "独立开发者的 AI 设计救星：impeccable"
date: "2026-04-09"
status: draft
tags:
  - claude-code
  - design
  - tools
  - indie-dev
published:
  公众号:
  知乎:
  博客:
---

你让 AI 帮你做了个界面，出来的东西大概长这样：

- 标题用 Inter，正文也用 Inter，按钮还是 Inter
- 主色调是蓝紫渐变，配上白色描边的玻璃拟态卡片
- 卡片里面套卡片，卡片里面还有卡片

你改了一版，换了个颜色，结果还是同款气质。换个提示词，依然如此。

这不是你的问题，也不是 AI 的 bug。

---

## 为什么 AI 画出来的界面都长一个样

所有大模型都从同一批数据学出来的——那批数据里充满了 Bootstrap 模板、Tailwind UI 示例、Dribbble 热门作品。AI 学会的是"什么设计看起来像设计"，而不是"什么设计适合这个产品"。

没有干预，AI 就会回归最安全的平均值。

impeccable 把这些典型反模式整理成了一个"Gallery of Shame"，起了专门的名字：

- **Inter Everywhere** — 没有理由地把 Inter 用遍全站
- **Cardocalypse** — 无止境地嵌套卡片
- **Purple Gradients** — 默认紫蓝渐变当主视觉
- **Side-Tab Cards** — 左侧粗色边框当成"有设计感"的捷径
- **Template Layouts** — 所有页面都像从同一个模板克隆的

认出来了吗？这就是你一直在和 AI 较劲的那些东西。

---

## impeccable 做了什么

impeccable 是一个开源工具包（Apache 2.0），给 AI 代码工具安装了一套设计知识库，同时提供 21 个设计指令。

用人话说：它让 AI 真的懂设计，然后给你一个共同语言，让你能精确指挥 AI 往哪个方向改。

不需要你懂设计，你只需要知道现在哪里有问题，然后挑对应的指令。

---

## 21 个 skill，怎么选

不用全学，按场景来。

### 先把基础装上

**`/teach-impeccable`**
一次性执行，把 impeccable 的设计知识加载进 AI 的上下文。之后所有对话都带着这套认知，不用每次重复交代。装完就忘，它在后台工作。

**`/audit`**
扫一遍你的界面，输出一份问题清单。不知道从哪里下手的时候，先跑这个。

### 日常打磨

| Skill | 干什么 |
|---|---|
| `/polish` | 全面质量收尾，对齐、间距、一致性 |
| `/typeset` | 修字体选择、字号层级、行距 |
| `/colorize` | 当界面太灰、太单调，加策略性的颜色 |
| `/arrange` | 修布局节奏，间距不统一、视觉流不顺 |
| `/clarify` | 改掉模糊的按钮文案、错误提示、标签 |
| `/quieter` | 界面太闹，视觉噪音太多，做减法 |
| `/adapt` | 移动端/桌面端适配 |

### 特殊场景

**`/overdrive`** — 想要出圈的视觉效果，让 AI 突破常规上限。做落地页、产品主页时用。

**`/distill`** — UI 堆了太多元素，用这个剥到核心。极简方向。

**`/onboard`** — 专门优化新用户引导流程、空状态、首次使用体验。

**`/animate`** — 加微交互和动效，让界面活起来。

**`/critique`** — 让 AI 从 UX 角度评审界面，指出信息层级、视觉引导的问题。不是帮你改，是帮你看清楚问题。

**`/bolder`** — 界面太保守太无聊，推一把让它更有个性。

---

## 怎么装

Claude Code 用户，两行命令选一个：

```bash
# 推荐，自动检测你的工具
npx skills add pbakaus/impeccable
```

或者直接在 Claude Code 里：

```
/plugin marketplace add pbakaus/impeccable
```

装完之后，第一步跑 `/teach-impeccable`，然后让 AI 帮你做个界面，再跑 `/audit` 看看它给出什么问题清单。

impeccable 还带了一个 CLI 工具可以扫 HTML/CSS/JSX 文件，以及一个 Chrome 插件可以在浏览器里实时检测。但作为独立开发者，从 Claude Code 里直接用 skill 已经够用了。

---

开源地址和完整 skill 文档在 [impeccable.style](https://impeccable.style)。
