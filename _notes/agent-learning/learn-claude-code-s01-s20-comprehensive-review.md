---
title: "learn-claude-code s01-s20：Comprehensive Agent 复习笔记"
date: 2026-06-28 13:01:37 +0800
tags: [Agent, Claude Code, Coding Agent, Comprehensive Agent, Tool Use, Memory, Worktree, MCP, Agent Teams]
main_category: "Agent"
sub_category: "learn-claude-code"
discipline: "Agent"
course: "learn-claude-code"
material_type: "综合复习"
description: "综合复习 learn-claude-code s01-s20：从最小 Agent Loop 到工具、权限、hooks、memory、team、worktree、MCP 和完整 coding agent harness。"
---

创建日期：2026-06-28  
适用范围：`s01_agent_loop` 到 `s20_comprehensive`  
目标：复习这个项目如何一步步从最小 Agent Loop 演进成完整 coding agent harness。

---

## 0. 一句话

参考：[learn-claude-code s01](https://learn.shareai.run/zh/s01/)

模型负责：

> 理解、推理、决定要不要调用工具、决定调用哪个工具。

Python 负责：

> 维护 `messages`、发送 API 请求、执行工具、检查权限、触发 hook、压缩上下文、保存状态、管理团队、接入外部工具，再把结果返回给模型。

所以整套系统不是“模型自己会操作电脑”，而是：

```text
模型输出 tool_use
Python 执行真实工具
Python 把 tool_result 放回 messages
模型继续基于结果推理
```

---

## 1. 总览：s01-s20 主线

这 20 节是在一步步搭建一个越来越完整的 Agent 系统：

```text
s01  最小 Agent Loop
s02  多工具分发
s03  权限检查
s04  Hooks 扩展点
s05  TodoWrite 计划状态
s06  Subagent 上下文隔离
s07  Skill Loading 按需加载能力
s08  Context Compact 压缩 messages
s09  Memory 长期记忆
s10  System Prompt 动态组装
s11  Error Recovery 错误恢复
s12  Task System 持久任务图
s13  Background Tasks 后台任务
s14  Cron Scheduler 定时任务
s15  Agent Teams 多 Agent 团队
s16  Team Protocols 请求响应协议
s17  Autonomous Agents 自治认领任务
s18  Worktree Isolation 工作目录隔离
s19  MCP Plugin 外部工具接入
s20  Comprehensive Agent 全部机制归入一个循环
```

最终主线仍然是 s01 的循环：

```text
用户输入
  -> messages
  -> client.messages.create(system, messages, tools)
  -> 模型返回 text 或 tool_use
  -> Python 执行工具
  -> tool_result 追加回 messages
  -> 下一轮
```

s20 的复杂性不是另一个“大脑”，而是围绕这个循环加了完整 harness。

---

## 2. 一页表格总览

| 模块 | 是什么 | 解决什么 | 为什么这样做 | 优点 | 代价 |
|---|---|---|---|---|---|
| s01 | 基础 Agent Loop | 让模型能请求工具、Python 能执行工具 | 把模型推理和真实执行分开 | 最小可运行闭环 | 工具少、安全弱 |
| s02 | 多工具系统 | 工具越来越多时如何分发 | 用 `TOOLS` 给模型看，用 `TOOL_HANDLERS` 给 Python 调用 | 扩展工具简单 | handler/schema 要保持一致 |
| s03 | 权限系统 | 阻止危险工具调用 | 安全边界必须在 Python 层 | 不完全依赖 prompt | 权限规则需要维护 |
| s04 | Hooks | 在循环关键点插入额外逻辑 | 把日志、权限、审计等从主循环拆出去 | 主循环更干净，可扩展 | 初学时抽象感强 |
| s05 | TodoWrite | 管理当前任务计划 | 让模型显式维护任务状态 | 复杂任务不容易跑偏 | 只适合当前会话 |
| s06 | Subagent | 隔离复杂子任务过程 | 子 Agent 用独立 `messages`，主 Agent 只收总结 | 节省主上下文 | 子过程细节默认不保留 |
| s07 | Skill Loading | 按需加载技能 | 先给轻量目录，需要时再加载全文 | 节省上下文 | 需要技能元信息和加载机制 |
| s08 | Context Compact | 压缩长 messages | 上下文窗口有限 | 防止 prompt 过长 | 细节可能丢失 |
| s09 | Memory | 长期记忆 | 压缩和跨会话都会丢信息 | 重要信息可长期复用 | 需要提取、筛选、维护 |
| s10 | System Prompt | 动态组装系统提示词 | system prompt 太长太乱 | 分区清晰，可按 context 装配 | 组装逻辑更复杂 |
| s11 | Error Recovery | API/上下文/输出截断恢复 | 不同错误要不同处理 | Agent 更稳定 | 恢复策略需要分类 |
| s12 | Task System | 持久任务图 | todo 只管当前会话，不够项目级 | 任务可依赖、可认领、可保存 | 状态管理更复杂 |
| s13 | Background Tasks | 慢任务后台跑 | 长命令会阻塞主循环 | 主循环可继续工作 | 后台结果要异步通知 |
| s14 | Cron Scheduler | 定时任务 | 未来时间触发任务 | 支持提醒和周期任务 | 需要队列和空闲处理 |
| s15 | Agent Teams | 多 Agent 协作 | 一个 Agent 做所有事效率低 | 队友独立上下文并行工作 | 通信和状态更复杂 |
| s16 | Team Protocols | 请求响应协议 | 队友通信需要匹配上下文 | `request_id` 避免混淆 | 需要维护 pending 状态 |
| s17 | Autonomous Agents | 队友自动找活 | Lead 不必手动分配每个任务 | 支持更长期自治 | 要防止重复认领 |
| s18 | Worktree Isolation | 工作目录隔离 | 多 Agent 同时改代码会冲突 | 每个任务可在独立 worktree 做 | Git/worktree 管理成本 |
| s19 | MCP Plugin | 外部工具接入 | 工具不能都写死在本地 | 通过协议动态发现工具 | 需要命名、权限、连接管理 |
| s20 | 完整 Agent | 全部机制合并 | 真实 Agent 需要同时具备这些能力 | 形成完整 harness | 代码复杂度最高 |

---

## 3. 架构分层

可以把整个系统分成 6 层：

```text
第 1 层：模型交互层
client.messages.create、messages、system、tools、tool_use、tool_result

第 2 层：工具执行层
TOOLS schema、TOOL_HANDLERS、bash/read/write/edit/glob/MCP tools

第 3 层：安全和扩展层
permission、hook、PreToolUse、PostToolUse、Stop

第 4 层：上下文和知识层
compact、memory、skills、system prompt sections

第 5 层：任务和协作层
todo_write、task graph、subagent、teammate、protocol、autonomous claim

第 6 层：运行时和外部集成层
background tasks、cron、worktree、MCP
```

面试或复习时，不要把每一节看成孤立功能。它们本质上都在回答同一个问题：

> 一个模型要长期、可靠、安全地操作真实环境，Python harness 需要帮它补哪些工程能力？

---

## 4. 每节复习卡片

### s01 Agent Loop

**是什么：** 最小 Agent 循环。  
**解决什么：** 模型不能直接执行命令，必须通过 Python 代理执行。  
**为什么这样做：** LLM 只负责输出结构化意图，真实副作用由程序控制。  
**核心流程：**

```text
user message -> LLM -> tool_use -> Python handler -> tool_result -> LLM
```

**优点：** 结构简单，是后续所有机制的基础。  
**缺点：** 工具少，没有权限、压缩、长期状态。

### s02 Tool Use

**是什么：** 多工具分发系统。  
**解决什么：** 当工具不止 `bash` 一个时，需要统一注册和调用。  
**为什么这样做：** 模型看 `TOOLS` schema，Python 用 `TOOL_HANDLERS` 分发。  
**核心理解：**

```text
TOOLS = 给模型看的工具说明
TOOL_HANDLERS = Python 真正调用的函数字典
```

**优点：** 新增工具时只要补 schema 和 handler。  
**缺点：** schema 和 handler 参数必须对应，否则运行时会报错。

### s03 Permission

**是什么：** 权限检查系统。  
**解决什么：** 模型可能请求危险命令，比如删除文件、系统操作。  
**为什么这样做：** 安全不能只写在 prompt 里，必须在 Python 执行前拦截。  
**核心理解：**

```text
模型可以提出危险工具调用
Python 可以拒绝执行
```

**优点：** 建立真实安全边界。  
**缺点：** deny list 和规则不可能天然完美，需要持续维护。

### s04 Hooks

**是什么：** 生命周期扩展点。  
**解决什么：** 日志、权限、审计、自动处理等逻辑如果都塞进主循环，会让主循环越来越乱。  
**为什么这样做：** 把“在某个时机要做的额外功能”注册到 hook 里。  
**核心理解：**

```text
hook = 字典 + 函数列表 + 在固定时机 for 循环调用
```

典型时机：

```text
UserPromptSubmit：用户输入后
PreToolUse：工具执行前
PostToolUse：工具执行后
Stop：本轮结束时
```

**优点：** 主循环保持稳定，新功能可以外挂。  
**缺点：** 调用链不再完全显式，初学时不容易跟踪。

### s05 TodoWrite

**是什么：** 当前会话内的计划和状态工具。  
**解决什么：** 复杂任务里模型容易忘记计划、跳步骤。  
**为什么这样做：** 让模型显式维护一个 todo list。  
**核心理解：**

```text
todo_write 不是文件工具
它是让模型管理当前任务计划的状态工具
```

**优点：** 复杂任务更有条理。  
**缺点：** 状态通常在内存中，不等于跨会话任务系统。

### s06 Subagent

**是什么：** 一次性子 Agent。  
**解决什么：** 复杂子任务会污染主 Agent 的 `messages`。  
**为什么这样做：** 子 Agent 用自己的 `messages` 完成任务，主 Agent 只拿最终总结。  
**核心理解：**

```text
主 Agent messages 只保存子 Agent 返回的总结
不保存子 Agent 读 10 个文件、跑 5 次命令的全部细节
```

**优点：** 上下文隔离，主线更清晰。  
**缺点：** 子过程细节默认丢失，除非主动记录。

### s07 Skill Loading

**是什么：** 技能按需加载。  
**解决什么：** 把所有技能全文都塞进 system prompt 会太长。  
**为什么这样做：** 先给模型轻量目录，需要时再 `load_skill(name)`。  
**核心链路：**

```text
SKILL.md
  -> _parse_frontmatter()
  -> SKILL_REGISTRY
  -> list_skills()
  -> build_system()
  -> client.messages.create(system=SYSTEM)
```

**优点：** 节省上下文，能力可扩展。  
**缺点：** 需要维护技能元信息和加载机制。

### s08 Context Compact

**是什么：** 上下文压缩。  
**解决什么：** `messages` 会越积越长，最终超过模型上下文窗口。  
**为什么这样做：** 在发送给模型前压缩旧内容，保留当前工作状态。  
**常见方式：**

```text
大 tool_result 持久化到文件
旧 tool_result 微压缩
中间 messages 裁剪
必要时让模型总结历史
```

**优点：** Agent 可以跑更久。  
**缺点：** 压缩一定可能损失细节。

### s09 Memory

**是什么：** 长期记忆系统。  
**解决什么：** 压缩会丢信息，换会话也会丢信息。  
**为什么这样做：** 把长期有价值的信息保存到 `.memory/`。  
**核心理解：**

```text
compact 主要解决当前 messages 太长
memory 主要解决跨会话长期信息保存
```

**优点：** 重要偏好、项目事实、长期约束可以复用。  
**缺点：** 需要判断什么值得记，错误记忆也会带来污染。

### s10 System Prompt

**是什么：** 动态 system prompt 组装。  
**解决什么：** 固定大 prompt 难维护，也不一定每轮都需要全部内容。  
**为什么这样做：** 把 prompt 拆成 sections，再根据 context 拼装。  
**核心理解：**

```text
assemble_system_prompt(context)
= identity + tools + workspace + memory + skills + MCP state ...
```

**优点：** 清晰、可维护、可按需注入。  
**缺点：** prompt 生成逻辑本身需要测试和维护。

### s11 Error Recovery

**是什么：** 错误恢复机制。  
**解决什么：** API 调用可能失败、输出可能截断、prompt 可能太长。  
**为什么这样做：** 不同错误原因不同，不能都用同一种重试。  
**核心分类：**

```text
429 / 529：等待后重试
max_tokens：先提高 max_tokens 再重试
prompt_too_long：压缩 messages 后重试
```

**优点：** Agent 更稳定。  
**缺点：** 恢复策略复杂，错误分类要准确。

### s12 Task System

**是什么：** 持久任务系统。  
**解决什么：** `todo_write` 只适合当前会话，不够项目级。  
**为什么这样做：** 用 `.tasks/*.json` 保存任务、状态、依赖、owner。  
**核心字段：**

```text
status：pending / in_progress / completed
owner：谁认领了
blockedBy：依赖哪些任务完成
```

**优点：** 支持长期任务、依赖、认领。  
**缺点：** 需要处理状态一致性，例如不能重复认领。

### s13 Background Tasks

**是什么：** 后台任务系统。  
**解决什么：** 慢操作会卡住主 Agent 循环。  
**为什么这样做：** 慢工具先返回 placeholder，真实结果后台完成后再通知。  
**核心流程：**

```text
慢 bash -> 后台线程执行
主循环立刻得到 tool_result: 已启动
后台完成 -> <task_notification> 注入 messages
```

**优点：** 主循环不被长任务阻塞。  
**缺点：** 结果变异步，模型需要处理延迟通知。

### s14 Cron Scheduler

**是什么：** 定时任务系统。  
**解决什么：** Agent 需要处理未来某个时间点发生的任务。  
**为什么这样做：** scheduler 只负责看时间，到点任务进入 queue，Agent 空闲时处理。  
**核心结构：**

```text
CronJob：任务定义
Scheduler：检查时间
cron_queue：保存已触发但未处理的任务
agent_loop：把任务变成 messages
```

**优点：** 支持提醒、周期任务，不直接打断主循环。  
**缺点：** Python 程序不运行时不会自动执行，只能保存 durable 定义。

### s15 Agent Teams

**是什么：** 多 Agent 团队。  
**解决什么：** 一个 Agent 做所有事情，容易慢且上下文混杂。  
**为什么这样做：** Lead Agent 可以 spawn teammate，每个 teammate 有自己的线程和 `messages`。  
**核心理解：**

```text
Lead 和 teammate 不共用 messages
通过 MessageBus / mailbox 交流
```

**优点：** 并行、隔离、角色分工。  
**缺点：** 通信、结果汇总、生命周期管理更复杂。

### s16 Team Protocols

**是什么：** 团队通信协议。  
**解决什么：** 普通消息不够表达请求、审批、响应匹配。  
**为什么这样做：** 用 `request_id` 和 `pending_requests` 管理请求响应。  
**核心理解：**

```text
request_id：把 response 对回原 request
pending_requests：内存里的待处理请求状态
.mailboxes/*.jsonl：消息传输记录
```

**优点：** 避免多个请求混在一起。  
**缺点：** 需要维护协议类型和状态机。

### s17 Autonomous Agents

**是什么：** 自治队友。  
**解决什么：** Lead 不想每个任务都手动派发。  
**为什么这样做：** 队友空闲时先检查 inbox，再扫描可认领任务。  
**核心流程：**

```text
idle_poll()
  -> read_inbox()
  -> scan_unclaimed_tasks()
  -> claim_task()
  -> 注入 <auto-claimed> 到 teammate messages
```

**优点：** 队友可以主动找活，系统更自动。  
**缺点：** 要严格检查 `owner`、`status`、`blockedBy`，防止抢同一个任务。

### s18 Worktree Isolation

**是什么：** Git worktree 工作目录隔离。  
**解决什么：** 多个 Agent 同时改同一个目录可能冲突。  
**为什么这样做：** 给任务绑定独立 worktree，队友认领后工具 cwd 切到该目录。  
**核心理解：**

```text
bind_task_to_worktree 只绑定目录
claim_task 才表示任务被某个 agent 认领
Python 必须真的把 bash/read/write 的 cwd 切过去
```

**优点：** 并行开发更安全，改动隔离。  
**缺点：** 需要管理分支、清理 worktree、防止误删未提交改动。

### s19 MCP Plugin

**是什么：** 外部工具接入协议。  
**解决什么：** 工具不能都写死在 Agent 本地代码里。  
**为什么这样做：** 外部 MCP server 发布工具，Agent 连接后动态发现。  
**核心流程：**

```text
connect_mcp("docs")
  -> MCPClient
  -> mcp_clients
  -> assemble_tool_pool()
  -> mcp__docs__search
```

**优点：** 外部能力可插拔，工具池动态扩展。  
**缺点：** 需要处理命名冲突、权限、连接状态、server 可靠性。

### s20 Comprehensive Agent

**是什么：** 完整 Agent harness。  
**解决什么：** 前 19 节每节单独讲机制，但真实 Agent 要同时具备这些能力。  
**为什么这样做：** 把所有组件挂回同一个 `agent_loop`。  
**核心理解：**

```text
机制很多，循环一个
```

s20 的一轮循环：

```text
注入 cron
注入 background notification
提醒 todo
prepare_context 压缩
update_context
assemble_tool_pool
call_llm
错误恢复
PreToolUse hook / permission
执行工具或后台任务
PostToolUse hook
追加 tool_result
继续下一轮
```

**优点：** 展示完整系统怎么运行。  
**缺点：** 代码量最大，理解时必须按主循环位置拆开看。

---

## 5. 易混淆的概念

### 5.1 `tools schema` vs `handler`

```text
tools schema：给模型看的工具格式，告诉模型工具名、参数类型、用途。
handler：Python 内部真正执行的函数。
```

模型不会直接拿到 Python 函数。  
模型只看到 schema，然后返回 `tool_use`。  
Python 根据 `tool_use.name` 找 handler。

### 5.2 `todo_write` vs `Task System`

```text
todo_write：当前会话内的轻量计划。
Task System：跨会话、可依赖、可认领、可保存的任务图。
```

### 5.3 `compact` vs `memory`

```text
compact：解决当前 messages 太长。
memory：保存长期有价值的信息，跨会话复用。
```

### 5.4 `subagent` vs `teammate`

```text
subagent：一次性，主 Agent 只拿最终总结。
teammate：长期线程，有自己的 inbox、messages、role，可以持续工作。
```

### 5.5 `mailbox` vs `pending_requests`

```text
mailbox：消息传输通道，存 JSONL 消息。
pending_requests：内存里的协议状态表，记录哪些请求还在等响应。
```

### 5.6 `bind_task_to_worktree` vs `claim_task`

```text
bind_task_to_worktree：这个任务应该在哪个目录做。
claim_task：这个任务由哪个 agent 负责，并进入 in_progress。
```

### 5.7 `connect_mcp` vs `assemble_tool_pool`

```text
connect_mcp：连接外部工具源。
assemble_tool_pool：把外部工具变成模型能看、Python 能调用的工具池。
```

---

## 6. s20 主循环图

```text
用户输入
  -> UserPromptSubmit hook
  -> messages.append(user query)
  -> agent_loop

agent_loop:
  -> consume_cron_queue()
  -> inject_background_notifications()
  -> todo reminder
  -> prepare_context()
  -> update_context()
  -> assemble_tool_pool()
  -> call_llm()
      -> with_retry()
      -> prompt_too_long reactive_compact()
      -> max_tokens recovery
  -> if no tool_use:
        Stop hook
        return
  -> for each tool_use:
        compact tool special case
        PreToolUse hook
        permission check
        maybe background dispatch
        handler(**input)
        PostToolUse hook
        collect tool_result
  -> messages.append(tool_results + background notifications)
  -> next round
```

---

## 7. 优点和代价总评

### 7.1 整体优点

- 模型和执行环境分工清晰。
- 工具系统可扩展。
- 权限边界在 Python 层，更可靠。
- hooks 让主循环可以扩展而不频繁改动。
- compact/memory/skills 让上下文更可控。
- task/team/worktree 支持更复杂的工程协作。
- background/cron 让 Agent 可以处理慢任务和未来任务。
- MCP 让外部工具接入变成协议问题。

### 7.2 整体代价

- 状态越来越多：`messages`、`context`、`tasks`、`memory`、`mailboxes`、`mcp_clients`、`background_tasks`。
- 调试更难：一次模型行为可能受 system prompt、memory、hook、permission、tools 等共同影响。
- 安全更复杂：本地工具、MCP 工具、后台任务、worktree 都需要权限边界。
- 一致性更难：任务状态、队友状态、协议状态、cron 状态都可能不同步。
- 压缩和记忆会影响模型认知：压缩可能丢细节，记忆可能引入旧信息。

---

## 8. 项目总结

```text
这个项目是在从零实现一个 coding agent harness。
最底层是 Agent Loop：模型基于 messages 和 tools schema 决定是否发起 tool_use，Python 根据 handler 执行工具，再把 tool_result 放回 messages。

后续章节不是改变这个核心循环，而是在循环周围补工程能力：
权限和 hook 负责安全与扩展；
todo、task、subagent、team 负责复杂任务管理；
compact、memory、skill、system prompt 负责上下文和能力管理；
background、cron、worktree、MCP 负责真实运行环境、并发协作和外部工具接入。

最终 s20 把这些机制合回一个主循环，所以核心可以总结为：
机制很多，循环一个。
```
