---
title: "learn-claude-code s01-s10：Quiz 标准答案复习笔记"
date: 2026-06-27 20:26:04 +0800
tags: [Agent, Claude Code, Coding Agent, Quiz, Tool Use, Hooks, Memory, Skill Loading, Context Compact]
main_category: "Agent"
sub_category: "learn-claude-code"
discipline: "Agent"
course: "learn-claude-code"
material_type: "Quiz 笔记"
description: "整理 learn-claude-code s01-s10 的 quiz 标准答案，覆盖 agent loop、多工具、权限、hooks、todo、subagent、skill loading、context compact、memory 和动态 system prompt。"
---

## 0. 总览

### s01-s10 的主线

这 10 节课是在一步步搭建一个越来越完整的 Agent 系统：

- s01：基础 Agent Loop，模型决定是否发起工具调用，Python 真正执行工具。
- s02：多工具系统，增加文件读写、编辑、glob，并用 handler 字典分发。
- s03：权限系统，在 Python 层阻止危险工具调用。
- s04：Hooks，把日志、检查、自动处理等扩展逻辑从主循环拆出去。
- s05：TodoWrite，让模型管理复杂任务的计划和状态。
- s06：Subagent，把复杂任务交给子 Agent，主 Agent 只接收总结结果。
- s07：Skill Loading，用轻量目录 + 按需加载的方式给模型扩展能力。
- s08：Context Compact，压缩当前 `messages`，防止上下文过长。
- s09：Memory，把长期有价值的信息保存到磁盘，跨会话复用。
- s10：System Prompt，把 system prompt 拆成 section，并根据 context 动态组装。

### 最重要的一句话

模型负责“理解、推理、决定要不要请求工具”；

Python 负责“维护 `messages`、发送 API 请求、执行工具、检查权限、触发 hook、保存状态、再把结果返回给模型”。

### 一条主线记忆

```text
s01 基础循环
-> s02 多工具
-> s03 权限检查
-> s04 hook 解耦
-> s05 todo 计划
-> s06 subagent 分工
-> s07 skill 能力加载
-> s08 compact 当前上下文
-> s09 memory 长期记忆
-> s10 system prompt 动态组装
```

## 1. s01 Agent Loop 的完整流程是什么？

用户输入先被放入 `messages`。Python 调用 `client.messages.create`，把 `messages`、`system`、`tools schema` 一起发给模型。

模型返回普通文本，或者返回 `tool_use`。如果返回 `tool_use`，Python 根据工具名找到真正的函数，例如 `run_bash`，然后执行工具。

工具执行完成后，Python 把结果包装成 `tool_result`，再放回 `messages`，然后再次调用模型，让模型基于工具结果继续回答。

核心结论：模型负责决定是否请求工具；Python 负责真正执行工具。

## 2. 为什么说模型拿到的是 tools schema，不是 Python 函数本身？

`tools schema` 是给模型看的工具说明书。它告诉模型：

- 工具叫什么
- 工具能做什么
- 工具需要哪些参数
- 参数是什么类型

但它不包含 Python 函数代码。模型看不到 `def run_bash(...)` 的实现，所以模型不能自己执行函数，只能返回一个工具调用请求。

## 3. TOOLS schema 和 TOOL_HANDLERS 的区别是什么？

`TOOLS schema` 是给模型看的工具菜单。

`TOOL_HANDLERS` 是给 Python 用的工具路由表。

模型返回类似“我要调用 `read_file`，参数是 `{"path": "a.txt"}`”的结构化请求。Python 再用工具名去 `TOOL_HANDLERS` 里找到真正的函数，并执行：

```python
handler = TOOL_HANDLERS.get(block.name)
output = handler(**block.input)
```

## 4. 为什么 s03 不能只靠 system prompt 做权限控制？

提示词只是劝模型遵守规则，Python 权限检查才是真正的门禁。

模型可能会请求危险命令，例如：

```bash
rm -rf /
```

如果没有 Python 层的 `DENY_LIST` 或 `check_permission`，程序可能真的执行危险操作。

所以安全边界必须放在程序里。模型可以提出工具调用请求，但 Python 必须在执行前判断是否允许。

## 5. hook 在 s04 的 agent loop 里解决了什么问题？

`hook` 把“循环过程中额外要做的事”从 `agent_loop` 主流程里拆出去。

如果没有 hook，每次想增加功能，比如工具调用前记录日志、工具调用后自动处理、停止时保存结果，都要修改 `agent_loop`。

有了 hook 后，主循环只需要在固定时机触发：

```python
trigger_hooks("PreToolUse", ...)
trigger_hooks("PostToolUse", ...)
trigger_hooks("Stop", ...)
```

具体执行什么逻辑，由注册进去的 hook 函数决定。

一句话：hook 是主循环关键节点预留给外部功能插入的位置。

## 6. hook 和 tool 的最大区别是什么？

`tool` 是模型能看见并请求调用的能力，例如 `bash`、`read_file`、`todo_write`。

`hook` 是 Python 内部自动触发的扩展点，模型通常看不见，也不会主动调用。

一句话：

```text
tool 给模型用，hook 给程序内部生命周期用。
```

## 7. s05 为什么要做 todo_write？

`todo_write` 用来让模型管理复杂任务的计划和状态。

它的作用不是读文件、写文件、执行命令，而是：

- 把复杂任务拆成待办事项
- 记录每一步的状态
- 让用户和程序看到当前进度
- 防止模型做着做着忘记任务

一句话：

```text
普通工具负责做事，todo_write 负责管理做事的计划。
```

## 8. 子 Agent 的详细工具调用会进入主 Agent 的 messages 吗？

不会。

如果主 Agent 调用 `task` 工具，子 Agent 内部读了 10 个文件、执行了 5 次 bash，这些详细过程只存在子 Agent 自己的临时循环里。

主 Agent 的 `messages` 里只保存 `task` 工具返回的最终总结字符串。

这样可以让主 Agent 的上下文更干净，避免被子 Agent 的大量中间过程撑爆。

## 9. SKILL.md 是怎么变成模型能看到的技能目录的？

完整链路是：

```text
SKILL.md 文件
-> _parse_frontmatter() 提取 name / description
-> _scan_skills() 扫描所有技能并登记
-> 存进 SKILL_REGISTRY
-> list_skills() 把字典转成普通文本目录
-> build_system() 把目录放进 system prompt
-> client.messages.create(system=SYSTEM) 发给模型
```

模型一开始看到的是轻量技能目录，不是所有完整 `SKILL.md`。

如果模型需要某个技能，再调用 `load_skill(name)`，Python 才把完整技能内容返回给模型。

## 10. 为什么不直接把所有 SKILL.md 都塞进 system prompt？

可以直接塞，但成本高，效果也可能变差。

主要问题是：

- 上下文太大
- 浪费 token
- 当前任务可能只需要一个技能
- 无关技能太多会干扰模型判断
- system prompt 变得难维护

所以 s07 使用渐进式披露：

```text
先给模型轻量技能目录
模型判断需要哪个技能
再用 load_skill 读取完整 SKILL.md
```

## 11. s08 为什么要做 context compact？

s08 是为了防止 `messages` 越积越多，工具结果越来越长，最后超过模型上下文限制，导致 API 报 `prompt_too_long`。

它主要处理：

- 大工具结果
- 老的 `tool_result`
- 中间历史消息
- 过长的对话上下文

目标是在尽量保留有用信息的前提下，把当前上下文压到模型能接受的范围内。

## 12. 为什么 s08 要先处理大的 tool_result，再做历史总结？

原则是：能便宜处理的先便宜处理，只有不够时才做昂贵且可能丢信息的总结压缩。

s08 的压缩层次大致是：

```text
1. persist_large_output
   把特别大的工具结果写到文件，只保留路径和预览

2. micro_compact
   把旧的长 tool_result 替换成占位信息

3. snip_compact
   裁掉中间一部分历史，只保留开头和最近几轮

4. compact_history
   让模型总结旧历史，用摘要替换原始消息
```

不一开始就总结，是因为总结会丢细节，也要额外调用模型，还可能总结错。

## 13. s08 compact 和 s09 memory 的区别是什么？

`s08 compact` 处理当前对话上下文太长的问题。

`s09 memory` 处理跨会话长期记忆的问题。

更具体地说：

```text
compact 处理 messages
memory 处理 .memory/*.md 文件
```

```text
compact 的目标：让当前上下文变短
memory 的目标：把用户偏好、项目事实、长期约束保存下来
```

## 14. s09 为什么要保存 pre_compress？

因为压缩后的 `messages` 可能已经丢掉细节。

如果直接从压缩后的内容里提取 memory，可能漏掉重要信息，例如用户偏好、项目事实、薄弱点、长期约束。

所以 s09 会在压缩前临时保存一份更完整的历史：

```python
pre_compress = messages.copy()
```

然后用这份更完整的 `pre_compress` 去提取长期记忆。

注意：它不是永久把整段超长对话塞回主上下文，而是在当前 Python 流程里临时使用。

## 15. s10 相比前面版本最大的变化是什么？

s10 不再只用一个写死的 `SYSTEM` 字符串，而是根据当前 `context` 动态组装 system prompt。

它把 prompt 分成多个 section，例如：

- `identity`
- `tools`
- `workspace`
- `memory`

然后根据当前上下文决定哪些 section 要加入最终 system prompt。

一句话：

```text
system prompt = 根据 context 动态拼出来的提示词。
```

## 16. 如果 .memory/MEMORY.md 不存在，memory section 会进入 system prompt 吗？

不会。

因为 s10 的逻辑大致是：

```python
memories = context.get("memories")
if memories:
    sections.append(memory_section)
```

在 Python 里，空字符串 `""`、`None` 都会被当成 False。

如果 `.memory/MEMORY.md` 不存在，`update_context()` 读不到 memory，`context["memories"]` 就会是空值，所以 `if memories` 不成立，memory section 不会加入最终 system prompt。

## 17. s01 到 s10 是怎么一步步演进的？

整体演进路线是：

```text
s01：基础 Agent Loop 和 bash 工具
s02：多工具系统和 TOOL_HANDLERS 分发
s03：权限检查
s04：hook 解耦扩展逻辑
s05：todo_write 计划管理
s06：task / subagent 分工
s07：skill loading，轻量目录 + 按需加载完整技能
s08：context compact，压缩当前 messages
s09：memory，保存跨会话长期记忆
s10：动态 system prompt，根据 context 拼装提示词
```

主线是：

```text
从“模型请求一个工具，Python 执行”
逐步扩展成
“安全、可扩展、有计划、可分工、能加载能力、能压缩上下文、能长期记忆、能动态组装提示词”的 Agent 系统。
```

## 18. s07 skill loading 和 s09 memory 的区别是什么？

一句话：

```text
skill = 我会什么
memory = 我记得什么
```

`skill loading` 加载的是能力说明，告诉模型某种任务应该怎么做、有哪些规则和步骤。

`memory` 加载的是过去记住的信息，告诉模型用户偏好、项目事实、长期约束等。

## 19. 每次工具调用结束后自动记录日志，应该用 tool、hook、memory 还是 system prompt？

应该用 `hook`，具体适合挂在 `PostToolUse hook`。

因为这个功能不是让模型主动调用的能力，也不是长期记忆，也不是提示词规则，而是每次工具调用结束后 Python 自动执行的一件事。

流程是：

```text
工具执行结束
-> trigger_hooks("PostToolUse", tool_name, result)
-> 日志 hook 自动记录工具名和结果长度
```

这正是 hook 的典型用途：在 agent loop 的固定节点插入额外程序逻辑，而不修改主循环。
