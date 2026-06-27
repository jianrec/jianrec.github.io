---
title: "AI Coding 发展史：从补全工具到多 Agent 指挥中心"
date: 2026-06-27 09:51:10 +0800
tags: [AI Coding, Coding Agent, Copilot, Claude Code, Codex, Gemini CLI, Agentic IDE, 开发工具]
main_category: "Agent"
sub_category: "AI Coding"
discipline: "Agent"
course: "AI Coding"
material_type: "技术脉络"
description: "整理 2021 到 2026 年 AI coding 的演进：从 IDE 补全、IDE Chat、agentic IDE，到 CLI agent 爆发和多 agent 桌面指挥中心。"
---

这几年 AI coding 的变化，不只是模型越来越强，而是开发者和 AI 的协作位置一直在变。

2021-2022，AI 主要是补代码；2023，AI 进入 IDE 对话窗口；2024，AI 开始读写项目和调用终端；2025，CLI agent 和 IDE agent 集中爆发；到 2026，问题已经变成：一个人怎么管理多个 agent 并行干活。

这条线索比单独看某个产品更清楚。

## 2021：AI 编程从“模型”进入 IDE

GUI 端最关键的节点是 GitHub Copilot technical preview。GitHub 在 2021 年 6 月推出 Copilot 技术预览，把 OpenAI 早期 Codex 代码模型接进 VS Code，让模型根据当前代码上下文补全一行代码、一个函数，甚至测试样例。

这个阶段的体验非常明确：

```text
开发者写代码
AI 看上下文
AI 补下一段
开发者决定是否接受
```

CLI 端这时还没成气候，主流体验仍是编辑器补全。

这一年的本质变化是：AI 不再只是回答编程问题，而是第一次大规模进入开发者每天写代码的地方。

## 2022：补全工具商业化，ChatGPT 打开大众认知

2022 年，GitHub Copilot GA，AI 代码补全开始正式收费和商业化。AWS 也推出 CodeWhisperer preview，把类似的代码推荐能力带进 IDE。

年底 ChatGPT 出现。它不是专门的 coding agent，但改变了开发者使用 AI 的方式：大家开始用自然语言让 AI 解释代码、写函数、改报错、总结文档。

这一阶段 AI 更像：

```text
高级自动补全 + 网页问答助手
```

GUI 端已经有清晰产品形态，CLI 端还没有真正爆发。

## 2023：IDE Chat 时代，开始能“问整个项目”

2023 年，AI coding 从补全转向聊天。

GitHub Copilot Chat 在 VS Code 和 Visual Studio 中 GA。Amazon CodeWhisperer GA。Google Duet AI for Developers 也在 2023 年 12 月 GA。Cursor 这类 AI-first 编辑器开始进入开发者视野。

这时你可以在 IDE 里问：

```text
这个函数干什么？
帮我改这段代码。
帮我写测试。
这个报错怎么修？
```

CLI 端，aider 这类 terminal pair programming 工具开始出现。GitHub Copilot in the CLI 也进入开发者工作流，但更偏命令解释、命令建议和局部辅助。

2023 的关键限制是：多数 AI IDE 还不能稳定地自动完成完整工程任务。它能帮你写、解释、局部修改，但通常不能自己读一堆文件、跑测试、看报错、循环修。

## 2024：Agentic IDE 萌芽，AI 开始动手改项目

2024 年是重要转折。

GUI 端开始出现真正的 agentic 形态。Cline 这类 VS Code agent 能读写文件、运行终端命令、使用浏览器，并在每一步要求用户确认。Windsurf 在 2024 年 11 月推出，强调 agentic IDE：AI 不只是聊天，而是能理解代码库、跨文件执行任务。

网页 GUI 也在变化。Claude Artifacts、ChatGPT Canvas 让用户在网页里预览和编辑代码，虽然它们不是完整本地 IDE，但让“对话 + 可编辑产物”的交互方式变得自然。

CLI 端，aider 等工具继续成熟，开始更明确地围绕本地 repo 修改、git diff、lint/test、提交变更等工作流展开。

这一年的本质变化是：

```text
AI 从“给建议”
变成“可以执行一部分开发动作”
```

但很多动作仍需要人确认。这个限制不是坏事，反而是 agentic coding 早期最重要的安全边界。

## 2025：CLI Agent 大爆发，GUI 和 CLI 合流

2025 年是 AI coding agent 真正爆发的一年。

CLI 端集中出现了几类代表：

- Claude Code GA，开发者可以在终端里委派工程任务。
- OpenAI Codex CLI 出现，Codex 这个名字从 2021 年的“代码模型”变成了本地终端 agent。
- Gemini CLI 发布，Google 也把 Gemini 带进终端。
- Amazon Q Developer 的 agentic coding 能力继续扩展，强调读写文件、生成 diff、运行 shell command，并结合 AWS 资源和开发环境。

GUI 端也同步升级：

- Copilot Agent Mode 进入 VS Code，可以多步改代码、读取相关文件、提出编辑、运行命令和测试，并根据错误循环修。
- Codex in ChatGPT 出现，成为云端 software engineering agent，可以在托管环境中处理写代码、调试、测试等任务。
- Codex IDE extension 进入 VS Code / Cursor / Windsurf 等 IDE。
- Kiro、Antigravity 这类 agentic IDE / agentic development platform 出现，强调 spec、任务、artifact、验证和多 agent 协作。
- Claude Code 也进入 VS Code / JetBrains 等 IDE 场景。

2025 年最重要的变化是：CLI 和 GUI 不再是两条完全不同的路。

同一个 agent 能在终端、IDE、网页、云端之间切换。AI 的能力从“帮你写代码”升级成“帮你完成任务”。

## 2026：桌面多 Agent 指挥中心

到 2026 年，重点开始从“单个 agent 能不能写代码”变成：

```text
怎么管理多个 agent？
怎么隔离它们的修改？
怎么让它们后台执行？
怎么审查它们产出的 diff？
怎么复用技能、自动化和上下文？
```

典型例子是 Codex App。OpenAI 在 2026 年 2 月推出 macOS 版 Codex app，2026 年 3 月更新支持 Windows。它不是单纯编辑器，而是一个 agent command center：可以让多个 agent 并行工作，用 worktree 隔离修改，设置自动化任务，并复用 skills。

2026 年的本质变化是：开发者不只是和一个 AI 聊天，而是在监督一组 agent 工作。

人的角色开始从：

```text
逐行写代码的人
```

部分转向：

```text
提出目标、拆解任务、审查 diff、决定合并的人
```

这不是说程序员不需要写代码了，而是写代码这件事的重心正在上移。

## 最清晰的总线

| 阶段 | 时间 | 主流形态 | AI 能力 |
| --- | --- | --- | --- |
| 补全时代 | 2021-2022 | Copilot、CodeWhisperer IDE 插件 | 补代码，不会自己跑 |
| 聊天时代 | 2023 | Cursor、Copilot Chat、IDE Chat | 解释代码、局部修改，基本不自动执行 |
| 初代 agent | 2024 | Cline、Windsurf、Canvas、Artifacts | 能读写文件、跑命令，但需要较多确认 |
| CLI agent 爆发 | 2025 | Claude Code、Codex CLI、Gemini CLI、Q CLI | 能规划、改文件、跑测试、循环修 |
| 多 agent 桌面 | 2026 | Codex App、Antigravity 等 | 并行任务、后台执行、自动化、指挥中心 |

压缩成一句话就是：

```text
2021-2022 是“AI 帮你补代码”；
2023 是“AI 在 IDE 里陪你聊代码”；
2024 是“AI 开始能动项目”；
2025 是“终端和 IDE agent 爆发”；
2026 是“人开始管理多个 AI agent 干活”。
```

## 我对这条线的理解

AI coding 的演进不是简单从“弱模型”到“强模型”。更准确地说，它经历了三个接口升级：

第一，输入接口升级。

从代码上下文补全，变成自然语言任务描述，再变成 issue、PR、测试输出、终端日志、浏览器状态等多源上下文。

第二，行动接口升级。

从只生成文本，变成读文件、写文件、执行命令、看测试结果、生成 diff、管理任务列表。

第三，协作接口升级。

从一个补全框，变成 IDE Chat，再变成 CLI agent，最后变成多 agent 指挥中心。

所以 AI coding 最值得关注的不是某个按钮，而是开发工作流的控制权怎么重新分配：

```text
模型负责更多执行
工具负责更多约束
人负责更高层判断
```

这也是为什么 coding agent 的核心不只是模型能力，还包括 permission、sandbox、worktree、diff review、test loop、memory、skills 和 automation。

真正有用的 AI coding，不是让 AI 无限制地“自动写”，而是让它在可审查、可回滚、可隔离的边界内完成更多真实工程动作。

## 参考资料

- [GitHub Copilot technical preview, 2021](https://github.blog/news-insights/product-news/introducing-github-copilot-ai-pair-programmer/)
- [GitHub Copilot GA, 2022](https://github.blog/news-insights/product-news/github-copilot-is-generally-available-to-all-developers/)
- [OpenAI ChatGPT, 2022](https://openai.com/index/chatgpt/)
- [AWS CodeWhisperer preview, 2022](https://aws.amazon.com/about-aws/whats-new/2022/06/aws-announces-amazon-codewhisperer-preview/)
- [Amazon CodeWhisperer GA, 2023](https://aws.amazon.com/about-aws/whats-new/2023/04/amazon-codewhisperer-generally-available/)
- [GitHub Copilot Chat GA, 2023](https://github.blog/news-insights/product-news/github-copilot-chat-now-generally-available-for-organizations-and-individuals/)
- [Google Duet AI for Developers GA, 2023](https://cloud.google.com/blog/products/ai-machine-learning/elevating-software-development-with-duet-ai-and-strategic-partners)
- [Cline overview](https://docs.cline.bot/cline-overview)
- [Windsurf launch context](https://devin.ai/blog/windsurf-rebrand-announcement/)
- [Claude Code GA with Claude 4, 2025](https://www.anthropic.com/news/claude-4)
- [OpenAI Codex and Codex CLI, 2025](https://openai.com/index/introducing-codex/)
- [Gemini CLI, 2025](https://blog.google/innovation-and-ai/technology/developers-tools/introducing-gemini-cli-open-source-ai-agent/)
- [Copilot Agent Mode in VS Code, 2025](https://code.visualstudio.com/blogs/2025/02/24/introducing-copilot-agent-mode)
- [Google Antigravity, 2025](https://developers.googleblog.com/build-with-google-antigravity-our-new-agentic-development-platform/)
- [OpenAI Codex App, 2026](https://openai.com/index/introducing-the-codex-app/)
