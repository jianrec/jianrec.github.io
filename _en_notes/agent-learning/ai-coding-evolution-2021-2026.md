---
title: "A History of AI Coding: From Completion Tools to a Multi-Agent Command Center"
date: 2026-06-27 09:51:10 +0800
tags: [AI Coding, Coding Agent, Copilot, Claude Code, Codex, Gemini CLI, Agentic IDE, Developer Tools]
main_category: "Paper Reading"
sub_category: "Agent · AI Coding"
discipline: "Agent"
course: "AI Coding"
material_type: "Technical Overview"
description: "A walkthrough of how AI coding evolved from 2021 to 2026: from IDE completion, IDE chat, and agentic IDEs, to the explosion of CLI agents and the multi-agent desktop command center."
lang: en
ref: "ai-coding-evolution-2021-2026"
---

The change in AI coding over the past few years is not just that models got stronger. It is that the position developers occupy when collaborating with AI has kept shifting.

In 2021-2022, AI mostly completed code. In 2023, AI moved into the IDE chat window. In 2024, AI started reading and writing projects and invoking the terminal. In 2025, CLI agents and IDE agents exploded together. By 2026, the question has become: how does one person manage multiple agents working in parallel?

This thread is clearer than looking at any single product in isolation.

## 2021: AI Programming Moves from "Model" into the IDE

The most important milestone on the GUI side is the GitHub Copilot technical preview. GitHub launched the Copilot technical preview in June 2021, wiring OpenAI's early Codex code model into VS Code so the model could complete a line, a function, or even a test case based on the current code context.

The experience at this stage was very clear:

```text
Developer writes code
AI reads the context
AI completes the next chunk
Developer decides whether to accept
```

The CLI side had not taken shape yet; the mainstream experience was still editor completion.

The essential change that year: AI was no longer just answering programming questions. For the first time, it entered at scale the place where developers write code every day.

## 2022: Completion Tools Go Commercial, ChatGPT Opens Up Public Awareness

In 2022, GitHub Copilot went GA, and AI code completion officially started charging and commercializing. AWS also launched the CodeWhisperer preview, bringing similar code recommendation capabilities into the IDE.

At the end of the year, ChatGPT appeared. It was not a dedicated coding agent, but it changed how developers used AI: people began using natural language to have AI explain code, write functions, fix errors, and summarize documentation.

At this stage AI was more like:

```text
Advanced autocomplete + a web Q&A assistant
```

The GUI side already had a clear product shape; the CLI side had not truly taken off.

## 2023: The IDE Chat Era, When You Could Start "Asking the Whole Project"

In 2023, AI coding shifted from completion to chat.

GitHub Copilot Chat went GA in VS Code and Visual Studio. Amazon CodeWhisperer went GA. Google Duet AI for Developers also went GA in December 2023. AI-first editors such as Cursor started entering developers' field of view.

At this point you could ask, inside the IDE:

```text
What does this function do?
Change this piece of code for me.
Write tests for me.
How do I fix this error?
```

On the CLI side, terminal pair programming tools such as aider began to appear. GitHub Copilot in the CLI also entered developer workflows, but leaned more toward command explanation, command suggestions, and localized assistance.

The key limitation of 2023: most AI IDEs still could not reliably complete a full engineering task on their own. They could help you write, explain, and make local edits, but usually could not read a pile of files, run tests, look at errors, and iterate on fixes by themselves.

## 2024: Agentic IDEs Emerge, AI Starts Modifying Projects

2024 was an important turning point.

On the GUI side, genuinely agentic forms began appearing. VS Code agents like Cline could read and write files, run terminal commands, and use the browser, asking the user to confirm at every step. Windsurf launched in November 2024, emphasizing the agentic IDE: AI is not just chatting, but can understand the codebase and execute tasks across files.

Web GUIs were changing too. Claude Artifacts and ChatGPT Canvas let users preview and edit code in the browser. They were not full local IDEs, but they made the "conversation plus editable artifact" interaction feel natural.

On the CLI side, tools like aider kept maturing and began organizing themselves more explicitly around workflows involving local repo modification, git diff, lint/test, and committing changes.

The essential change that year:

```text
AI moved from "offering suggestions"
to "being able to execute some development actions"
```

But many actions still required human confirmation. This limitation was not a bad thing; it was the most important safety boundary in the early days of agentic coding.

## 2025: The CLI Agent Explosion, GUI and CLI Converge

2025 was the year AI coding agents truly exploded.

Several representative examples emerged on the CLI side at once:

- Claude Code went GA, letting developers delegate engineering tasks from the terminal.
- OpenAI Codex CLI appeared, and the name Codex shifted from a 2021 "code model" to a local terminal agent.
- Gemini CLI launched, bringing Gemini into the terminal for Google as well.
- Amazon Q Developer's agentic coding capabilities kept expanding, emphasizing reading and writing files, generating diffs, and running shell commands, integrated with AWS resources and development environments.

The GUI side upgraded in parallel:

- Copilot Agent Mode arrived in VS Code, able to change code across multiple steps, read relevant files, propose edits, run commands and tests, and iterate on fixes based on errors.
- Codex in ChatGPT appeared as a cloud software engineering agent that could handle writing code, debugging, and testing in a hosted environment.
- The Codex IDE extension arrived in IDEs like VS Code, Cursor, and Windsurf.
- Agentic IDEs and agentic development platforms like Kiro and Antigravity appeared, emphasizing specs, tasks, artifacts, validation, and multi-agent collaboration.
- Claude Code also moved into IDE contexts such as VS Code and JetBrains.

The most important change in 2025: CLI and GUI were no longer two entirely separate paths.

The same agent could move between terminal, IDE, web, and cloud. AI's capability upgraded from "helping you write code" to "helping you complete tasks."

## 2026: The Desktop Multi-Agent Command Center

By 2026, the focus started shifting from "can a single agent write code" to:

```text
How do you manage multiple agents?
How do you isolate their changes?
How do you let them run in the background?
How do you review the diffs they produce?
How do you reuse skills, automation, and context?
```

The archetypal example is the Codex App. OpenAI launched the macOS version of the Codex app in February 2026, with a March 2026 update adding Windows support. It is not just an editor but an agent command center: it lets multiple agents work in parallel, isolates changes with worktrees, sets up automated tasks, and reuses skills.

The essential change in 2026: developers are not just chatting with one AI; they are supervising a group of agents at work.

The human role starts moving from:

```text
the person who writes code line by line
```

partly toward:

```text
the person who sets goals, breaks down tasks, reviews diffs, and decides what to merge
```

This does not mean programmers no longer need to write code. It means the center of gravity of writing code is moving up a level.

## The Clearest Through-Line

| Stage | Time | Mainstream Form | AI Capability |
| --- | --- | --- | --- |
| The completion era | 2021-2022 | Copilot, CodeWhisperer IDE plugins | Completes code, cannot run on its own |
| The chat era | 2023 | Cursor, Copilot Chat, IDE Chat | Explains code, makes local edits, barely executes automatically |
| First-generation agents | 2024 | Cline, Windsurf, Canvas, Artifacts | Can read/write files and run commands, but needs frequent confirmation |
| The CLI agent explosion | 2025 | Claude Code, Codex CLI, Gemini CLI, Q CLI | Can plan, edit files, run tests, iterate on fixes |
| Multi-agent desktop | 2026 | Codex App, Antigravity, etc. | Parallel tasks, background execution, automation, command center |

Compressed into one sentence:

```text
2021-2022 was "AI helps you complete code";
2023 was "AI chats with you about code inside the IDE";
2024 was "AI starts being able to act on the project";
2025 was "the explosion of terminal and IDE agents";
2026 is "people start managing multiple AI agents doing the work".
```

## How I Read This Thread

The evolution of AI coding is not simply a move from "weak models" to "strong models." More precisely, it went through three interface upgrades.

First, the input interface upgraded.

From code-context completion, to natural-language task descriptions, to multi-source context such as issues, PRs, test output, terminal logs, and browser state.

Second, the action interface upgraded.

From generating text only, to reading files, writing files, executing commands, inspecting test results, generating diffs, and managing task lists.

Third, the collaboration interface upgraded.

From a single completion box, to IDE chat, then to CLI agents, and finally to a multi-agent command center.

So the most noteworthy thing about AI coding is not any particular button, but how control over the development workflow gets redistributed:

```text
The model takes on more execution
The tooling takes on more constraints
Humans take on higher-level judgment
```

That is also why the core of a coding agent is not just model capability, but also permission, sandbox, worktree, diff review, test loop, memory, skills, and automation.

Genuinely useful AI coding is not about letting AI "auto-write" without limits. It is about letting it complete more real engineering actions within boundaries that are reviewable, revertible, and isolated.

## References

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