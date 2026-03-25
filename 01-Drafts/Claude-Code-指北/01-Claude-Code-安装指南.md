---
series: Claude Code 指北
title: "国内怎么用上 Claude Code？Mac + Windows 从零到跑通"
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

Claude Code 是目前最强的 AI 编程助手，但 Anthropic 官方不对大陆提供服务。注册要海外手机号，付费要海外信用卡，API 还得翻墙——光是装上就劝退了一大批人。

今天这篇帮你彻底解决这个问题。用的是 [AI Code Mirror](https://www.aicodemirror.com/register?invitecode=03JCSO) 这个方案，**安装包是官方原版**，只是 API 走了国内可用的通道，体验和官方完全一致。我试过好几家镜像站，这家延迟最低、同步最快、到现在没断过服务。

不管你是 Mac 还是 Windows，不管有没有编程经验，跟着下面的步骤来就行。

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

密钥长这样：`sk-ant-xxxx...`，一长串字符。复制的时候注意别漏了前后的字符。

拿到密钥后，根据你的电脑系统，往下找对应的教程。

---

## Mac 用户看这里

### 第二步：打开终端

终端是什么？就是一个可以输入命令的窗口。Mac 自带了一个。

打开方式：按 `Cmd + 空格`，输入 `Terminal`（或者"终端"），回车。

看到一个黑色（或白色）的窗口弹出来了？这就是终端。接下来的命令都在这里输入。

### 第三步：运行环境准备脚本

在终端里输入：

```bash
curl -fsSL https://download.aicodemirror.com/env_deploy/env-install.sh | bash
```

这条命令会自动帮你搞定运行环境：检测系统、安装 Node.js（Claude Code 的基础依赖）、配好国内 npm 镜像源。全自动，跑完会告诉你哪些通过了、哪些没通过。

> 如果你之前装过旧版 Claude Code，先跑一下 `npm uninstall -g @anthropic-ai/claude-code` 卸掉，再继续下面的步骤。

### 第四步：安装 Claude Code

在终端里输入：

```bash
npm install -g @anthropic-ai/claude-code
```

> `npm` 是上一步自动装好的包管理工具。这行命令的意思就是"把 Claude Code 装到我电脑上"。

等它跑完，不报红色错误就是成功了。

### 第五步：配置密钥

还记得第一步复制的那个密钥吗？现在用上了。在终端里输入：

```bash
curl -fsSL https://download.aicodemirror.com/env_deploy/env-deploy.sh | bash -s -- "把这里替换成你的密钥"
```

注意引号要保留，只替换中间的文字。

### 第六步：验证

**关掉终端，重新打开**（这步很重要，别跳过），然后输入：

```bash
claude -v
```

看到版本号了？恭喜，装好了！直接跳到最后的「开始使用」。

---

## Windows 用户看这里

### 第二步：安装 Git 和 Node.js

需要装两个东西，都是一路点下一步，不要改路径：

1. **Git**（代码管理工具，Mac 自带但 Windows 没有）：[git-scm.com/downloads/win](https://git-scm.com/downloads/win)
2. **Node.js**（Claude Code 的运行环境）：[nodejs.org/zh-cn/download](https://nodejs.org/zh-cn/download)

两个都装完后，打开 **Windows PowerShell**（开始菜单搜索 `PowerShell`，认准**蓝色图标**），输入以下命令确认装好了：

```powershell
node -v
npm -v
```

两个都能显示版本号就行。

> 如果提示「No suitable shell found」，说明 Git 没装好。在系统环境变量里加一个变量：变量名 `CLAUDE_CODE_GIT_BASH_PATH`，变量值 `C:\Program Files\git\bin\bash.exe`（怎么加环境变量看第四步的说明）。加完后关掉 PowerShell 重新打开再试。

### 第三步：安装 Claude Code

在 PowerShell 里输入：

```powershell
npm install -g @anthropic-ai/claude-code
```

等它跑完，不报红色错误就是成功了。

> 如果之前装过旧版，先跑一下 `npm uninstall -g @anthropic-ai/claude-code` 卸掉再装。

### 第四步：配置密钥

Windows 不能用一行命令配置密钥，需要手动设置「环境变量」。听起来复杂，其实就是告诉你的电脑"以后用 Claude Code 的时候，自动带上这个密钥"。

**怎么打开：** 点击 Windows 开始菜单，搜索「编辑系统环境变量」，打开它。弹出「系统属性」窗口后，点击右下角的「环境变量」按钮。

在**下方的「系统变量」区域**，点「新建」，依次添加这三个变量：

| 变量名 | 变量值 |
|--------|--------|
| `ANTHROPIC_BASE_URL` | `https://api.aicodemirror.com/api/claudecode` |
| `ANTHROPIC_API_KEY` | 你的密钥 |
| `ANTHROPIC_AUTH_TOKEN` | 你的密钥 |

> 每次新建之前，先检查用户变量和系统变量里有没有同名的旧变量，有的话先删掉再建新的。

三个都加完后，一路点确定关掉窗口。

### 第五步：验证

**关掉 PowerShell，重新打开**，输入：

```powershell
claude -v
```

看到版本号了？恭喜，装好了！

---

## 装不上？看这里

几个最常见的坑：

**npm install 报权限错误（Mac）**

提示 `EACCES` 或 `permission denied`？不要用 `sudo`，改用这个命令重新设置 npm 的全局安装路径：

```bash
mkdir -p ~/.npm-global && npm config set prefix '~/.npm-global' && echo 'export PATH=~/.npm-global/bin:$PATH' >> ~/.zshrc && source ~/.zshrc
```

然后重新跑 `npm install -g @anthropic-ai/claude-code`。

**npm install 卡住不动**

大概率是网络问题。换成国内镜像源试试：

```bash
npm config set registry https://registry.npmmirror.com
```

设完之后重新安装。

**PowerShell 提示"无法加载文件，因为在此系统上禁止运行脚本"（Windows）**

用管理员身份打开 PowerShell，输入：

```powershell
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser
```

输入 `Y` 确认，然后重试。

**`claude` 命令提示 not found**

先确认你装完后**关掉终端重新打开了**。如果还不行，检查 npm 全局安装路径是否在系统 PATH 里。输入 `npm bin -g` 看一下路径，把它加到环境变量的 PATH 中。

**连上了但报 401 / API key 无效**

检查密钥有没有复制完整（前后不要有空格），环境变量名有没有拼对。Windows 用户改完环境变量后必须重新打开 PowerShell 才生效。

如果还不行（Windows），试试这几步：
1. 备份 `C:\Users\你的用户名\.claude.json` 文件
2. 删掉这个文件
3. 重新打开 Claude Code，弹出的交互页选 `yes`

**提示 Unable to connect to Anthropic services（Windows）**

跟上面一样，先检查环境变量。如果确认没问题，打开 `C:\Users\你的用户名\.claude.json`，在最外层 JSON 里加一行 `"hasCompletedOnboarding": true`，保存后重启 PowerShell 再试。

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

