---
series: Claude Code实战指南
title: "国内怎么用上 Claude Code？保姆级安装指南"
date: "2026-03-24"
status: draft
tags:
  - claude-code
  - 入门
published:
  公众号: 
  知乎: 
  小红书: 
  X: 
  博客: 
---

朋友们好，我是 Walker。

这是「Claude Code 实战指南」的第一篇。

先说一个现实：Claude Code 是目前最强的 AI 编程助手，但 Anthropic 官方不对大陆提供服务。注册要海外手机号，付费要海外信用卡，API 还得翻墙——光是装上就劝退了一大批人。

今天这篇帮你彻底解决这个问题。用的是 [AI Code Mirror](https://www.aicodemirror.com/register?invitecode=03JCSO) 这个方案，**安装包是官方原版**，只是 API 走了国内可用的通道。体验和官方完全一致。

> AI Code Mirror 算是我用了很多家镜像站里最稳的

不管你是 Mac 用户还是 Windows 用户，不管你有没有编程经验，跟着下面的步骤一步步来就行。

## Claude Code 是什么

你可能用过 ChatGPT 的网页版——把代码贴过去问它，它给你答案，你再贴回来。

Claude Code 不一样。**它直接跑在你电脑上，活在你的项目文件夹里。** 你跟它说"帮我改一下这个文件"，它真的会帮你改。你说"帮我跑一下测试"，它真的会帮你跑。

不用复制粘贴，不用来回切换窗口。

我现在写代码、写文章、搞自动化，基本都靠它。后面的系列文章会详细讲怎么用，但首先你得装上。

## 第一步：注册 AI Code Mirror，拿到密钥

不管你是什么系统，这一步都一样。

1. 打开 [aicodemirror.com](https://www.aicodemirror.com/register?invitecode=03JCSO)，注册一个账号
2. 登录后，找到「API 密钥」页面
3. 点击创建一个新的密钥（Key）
4. 把生成的密钥**复制下来**，保存好，后面要用

<!-- 📸 截图：AI Code Mirror 的 API 密钥页面，标注"创建密钥"按钮 -->

拿到密钥后，根据你的电脑系统，往下找对应的教程。

---

## Mac 用户看这里

### 第二步：打开终端

终端是什么？就是一个可以输入命令的窗口。Mac 自带了一个。

打开方式：按 `Cmd + 空格`，输入 `Terminal`（或者"终端"），回车。

<!-- 📸 截图：Spotlight 搜索 Terminal -->

看到一个黑色（或白色）的窗口弹出来了？这就是终端。接下来的命令都在这里输入。

### 第三步：安装 Node.js

Node.js 是 Claude Code 运行需要的基础环境，类似于"你得先装个播放器才能看视频"。

在终端里输入以下命令（复制粘贴进去，按回车）：

```bash
node --version
```

如果显示了一个 `v18` 或更高的版本号（比如 `v22.14.0`），说明你已经装好了，跳到第四步。

如果提示 `command not found` 或者版本号低于 18，去 [nodejs.org/zh-cn/download](https://nodejs.org/zh-cn/download) 下载安装包，打开后一路点下一步就行。

<!-- 📸 截图：Node.js 官网下载页面，标注 macOS 安装包 -->

装完后**关掉终端再重新打开**，再试一次 `node --version`，确认有版本号了。

### 第四步：运行环境检查

在终端里输入：

```bash
curl -fsSL https://download.aicodemirror.com/env_deploy/env-install.sh | bash
```

这条命令会自动检查你的电脑环境，确保一切准备就绪。等它跑完就行。

### 第五步：安装 Claude Code

在终端里输入：

```bash
npm install -g @anthropic-ai/claude-code
```

> `npm` 是刚才装 Node.js 时自动带的一个工具，专门用来安装软件包。这行命令的意思就是"把 Claude Code 装到我电脑上"。

等它跑完，不报红色错误就是成功了。

### 第六步：配置密钥

还记得第一步复制的那个密钥吗？现在用上了。在终端里输入：

```bash
curl -fsSL https://download.aicodemirror.com/env_deploy/env-deploy.sh | bash -s -- "把这里替换成你的密钥"
```

注意引号要保留，只替换中间的文字。

### 第七步：验证

**关掉终端，重新打开**（这步很重要，别跳过），然后输入：

```bash
claude -v
```

看到版本号了？恭喜，装好了！直接跳到最后的「开始使用」。

---

## Windows 用户看这里

### 第二步：安装 Git

Git 是一个代码管理工具，Claude Code 需要它才能正常工作。

去 [git-scm.com/downloads/win](https://git-scm.com/downloads/win) 下载，打开安装包后**一路点下一步**，不要修改任何路径设置。

<!-- 📸 截图：Git 安装界面 -->

### 第三步：安装 Node.js

和 Mac 一样，Node.js 是 Claude Code 的基础运行环境。

去 [nodejs.org/zh-cn/download](https://nodejs.org/zh-cn/download) 下载，同样一路点下一步。

<!-- 📸 截图：Node.js 官网下载页面，标注 Windows 安装包 -->

### 第四步：打开 PowerShell

PowerShell 是 Windows 上的终端（输入命令的窗口）。

打开方式：点击 Windows 开始菜单，搜索 `PowerShell`，打开那个**蓝色图标**的 Windows PowerShell。

<!-- 📸 截图：搜索 PowerShell -->

### 第五步：确认 Git 和 Node 装好了

在 PowerShell 里依次输入：

```powershell
node -v
npm -v
```

两个都能显示版本号就行。

> 如果提示「No suitable shell found」，说明 Git 没装好。在系统环境变量里加一个变量：变量名 `CLAUDE_CODE_GIT_BASH_PATH`，变量值 `C:\Program Files\git\bin\bash.exe`（怎么加环境变量看第七步的说明）。加完后关掉 PowerShell 重新打开再试。

### 第六步：安装 Claude Code

在 PowerShell 里输入：

```powershell
npm install -g @anthropic-ai/claude-code
```

等它跑完，不报红色错误就是成功了。

### 第七步：配置密钥

Windows 不能用一行命令配置密钥，需要手动设置「环境变量」。听起来复杂，其实就是告诉你的电脑"以后用 Claude Code 的时候，自动带上这个密钥"。

**怎么打开：** 点击 Windows 开始菜单，搜索「编辑系统环境变量」，打开它。

<!-- 📸 截图：搜索"编辑系统环境变量" -->

弹出的窗口里，点击右下角的「环境变量」按钮。

<!-- 📸 截图：系统属性窗口，标注"环境变量"按钮 -->

在**下方的「系统变量」区域**，点「新建」，依次添加这三个变量：

| 变量名 | 变量值 |
|--------|--------|
| `ANTHROPIC_BASE_URL` | `https://api.aicodemirror.com/api/claudecode` |
| `ANTHROPIC_API_KEY` | 你的密钥 |
| `ANTHROPIC_AUTH_TOKEN` | 你的密钥 |

<!-- 📸 截图：新建环境变量的对话框，标注变量名和变量值 -->

> 每次新建之前，先上下看看有没有同名的旧变量，有的话先删掉再建新的。

三个都加完后，一路点确定关掉窗口。

### 第八步：验证

**关掉 PowerShell，重新打开**，输入：

```powershell
claude -v
```

看到版本号了？恭喜，装好了！

---

## 开始使用

现在来试一下。在终端（Mac）或 PowerShell（Windows）里，进入你电脑上任意一个有代码的文件夹：

```bash
cd 你的项目路径
```

> 比如你的项目在桌面上的 my-project 文件夹里，Mac 输入 `cd ~/Desktop/my-project`，Windows 输入 `cd C:\Users\你的用户名\Desktop\my-project`

然后启动 Claude Code：

```bash
claude
```

试着跟它说一句：

```
帮我看看这个项目的结构
```

它开始分析你的文件了？恭喜，你已经拥有了目前最强的 AI 编程助手。

装上只是开始。Claude Code 裸装就能用，但要真正好用，还差几步配置——CLAUDE.md、权限模式、还有一些能让效率翻倍的技巧。

下一篇见。

