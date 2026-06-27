---
title: "learn-claude-code s11-s15：从容错到多 Agent 团队"
date: 2026-06-27 21:44:51 +0800
tags: [Agent, Claude Code, Coding Agent, Error Recovery, Task System, Background Tasks, Cron Scheduler, Agent Teams]
main_category: "Agent"
sub_category: "learn-claude-code"
discipline: "Agent"
course: "learn-claude-code"
material_type: "源码学习"
description: "复习 learn-claude-code s11-s15：Error Recovery、Task System、Background Tasks、Cron Scheduler 和 Agent Teams。"
---

## 0. 总览

### s11-s15 的主线

这 5 节课是在把前面的单 Agent loop，继续升级成更可靠、更能长期运行、更能协作的系统：

- s11：Error Recovery，让 LLM/API 调用失败时可以分类恢复，而不是直接崩溃。
- s12：Task System，把大目标拆成可持久化、有依赖、可认领的任务图。
- s13：Background Tasks，把慢工具放到后台线程，主 Agent 继续工作。
- s14：Cron Scheduler，让 Agent 可以按时间表自动产生工作。
- s15：Agent Teams，让 Lead Agent 可以创建队友，通过 mailbox 协作。

### 最重要的一句话

模型只能看到 `messages`。所以工具结果、后台通知、定时任务、队友消息，最后都要变成 `messages` 的一部分，模型才能继续推理。

### 一条主线记忆

```text
s11：调用失败怎么恢复
-> s12：大任务怎么持久化管理
-> s13：慢工具怎么不阻塞主循环
-> s14：未来时间点怎么自动触发任务
-> s15：多个 Agent 怎么协作
```

## 1. s11 Error Recovery

### 解决什么问题

s11 解决的是：

```text
Agent 遇到 API 错误、输出截断、上下文过长时，不要直接崩溃。
```

s11 相比 s10 的核心变化是：

```text
s10：正常调用模型
s11：模型调用外面包一层恢复逻辑
```

### 三条恢复路径

#### 1. max_tokens：输出被截断

触发条件：

```python
response.stop_reason == "max_tokens"
```

含义：

```text
模型不是失败了，而是输出额度不够，话还没说完。
```

恢复策略：

```text
第一次 max_tokens：
    不把截断输出 append 到 messages
    把 max_tokens 从 8000 升到 64000
    用同样的 messages 重新请求

如果 64000 还是不够：
    保存截断输出
    注入 continuation prompt
    让模型从断点继续说
```

为什么第一次不 append 截断输出？

```text
因为第一次只是输出纸不够大。
最干净的做法是原始请求不变，只换一张更大的答题纸重新生成。
```

#### 2. prompt_too_long：输入上下文太长

含义：

```text
messages + system + tools 太长，模型上下文装不下。
```

恢复策略：

```text
reactive_compact(messages)
压缩 messages
然后重试
```

为什么不能等一会儿再试？

```text
因为请求内容本身太长，等待不会让它变短。
必须修改请求内容，也就是压缩 messages。
```

#### 3. 429 / 529：临时服务错误

429：

```text
Too Many Requests / Rate Limit
请求太频繁，被限流。
```

529：

```text
Overloaded
服务端过载，暂时处理不了。
```

恢复策略：

```text
with_retry()
指数退避 + 随机抖动
等待后重试同一个请求
```

为什么 429/529 不压缩 messages？

```text
因为 messages 本身没问题。
问题在服务端暂时忙，等一会儿可能就好了。
```

### s11 核心对比

| 错误 | 原因 | 处理方式 |
|---|---|---|
| `max_tokens` | 输出额度不够 | 先升 max_tokens，再续写 |
| `prompt_too_long` | 输入上下文太长 | 压缩 `messages` |
| `429` | 请求太频繁 | 等待重试 |
| `529` | 服务端过载 | 等待重试，必要时切备用模型 |

### s11 记忆句

```text
max_tokens：输出纸不够，换大纸。
prompt_too_long：输入太长，压 messages。
429/529：服务暂时不接，等一等。
```

## 2. s12 Task System

### 解决什么问题

s05 的 `todo_write` 是当前会话里的临时计划，适合管理：

```text
我现在要做哪几步
```

s12 的 Task System 是长期任务系统，适合管理：

```text
一个大项目拆成哪些任务
哪些任务依赖哪些任务
哪个任务正在做
程序重启后怎么恢复进度
```

### 核心数据结构：Task

```python
@dataclass
class Task:
    id: str
    subject: str
    description: str
    status: str
    owner: str | None
    blockedBy: list[str]
```

字段含义：

- `id`：任务唯一 ID。
- `subject`：任务标题。
- `description`：任务详细描述。
- `status`：`pending` / `in_progress` / `completed`。
- `owner`：谁认领了这个任务。
- `blockedBy`：当前任务被哪些前置任务阻塞。

### blockedBy 是什么

`blockedBy` 表示：

```text
我依赖哪些任务，这些任务没完成前我不能开始。
```

例子：

```text
task_1：设计数据库表
task_2：写 API，blockedBy=["task_1"]
task_3：写测试，blockedBy=["task_2"]
```

意思是：

```text
task_2 必须等 task_1 completed 后才能开始。
task_3 必须等 task_2 completed 后才能开始。
```

### claim_task 是什么

`claim_task` 表示：

```text
认领任务 / 开始接这个任务
```

状态变化：

```text
pending -> in_progress
```

同时设置：

```text
owner = "agent"
```

### 为什么 claim_task 前要 can_start

因为不能跳过依赖。

`can_start(task_id)` 会检查：

```text
blockedBy 里的所有任务是否都 completed
```

如果前置任务没完成，就不能认领当前任务。

### Task System 和 todo_write 的区别

| 对比 | todo_write | Task System |
|---|---|---|
| 位置 | s05 | s12 |
| 作用 | 当前对话里的计划 | 长期项目任务图 |
| 存储 | 内存 / messages | `.tasks/*.json` |
| 是否跨会话 | 不适合 | 可以 |
| 是否有依赖 | 没有 | 有 `blockedBy` |
| 是否有 owner | 没有 | 有 |
| 适合场景 | 当前任务怎么做 | 大项目怎么推进 |

### s12 记忆句

```text
todo_write 是便签纸；Task System 是项目管理看板。
```

## 3. s13 Background Tasks

### 解决什么问题

s13 解决的是：

```text
慢工具不要阻塞 Agent 主循环。
```

比如：

```text
npm install
pytest
docker build
```

这些命令可能跑很久。如果同步执行，Agent 只能等命令返回，不能继续做别的事。

### 谁在后台跑

后台跑的是：

```text
Python 开出来的后台线程
```

代码模式：

```python
threading.Thread(target=worker, daemon=True).start()
```

后台线程里真正执行：

```python
result = execute_tool(block)
```

如果工具是 bash，那么后台线程就在跑 bash 命令。

注意：

```text
不是模型在后台跑。
不是 Agent 大脑在后台跑。
是 Python 线程在后台跑慢工具。
```

### Agent 主循环在干什么

Agent 主循环不用等慢命令完成。它会立刻返回一个占位 `tool_result`：

```text
Background task bg_0001 started.
Result will be available when complete.
```

然后模型可以继续请求其他工具，比如：

```text
read_file
list_tasks
write_file
```

### 为什么启动后台任务后也要立刻返回 tool_result

因为 Messages API 规则是：

```text
每一个 tool_use 都必须有一个对应的 tool_result。
```

后台任务虽然没完成，但这次工具请求必须先被“回应”。

所以返回的是占位结果：

```text
这个后台任务已经启动，最终结果稍后通知。
```

### 为什么完成后用 task_notification

启动时已经返回过一次：

```text
tool_use_id: abc123
tool_result: Background task bg_0001 started
```

这个 `tool_use` 和 `tool_result` 的配对已经完成。

后台完成后不能再用同一个 `tool_use_id` 返回第二个 `tool_result`，否则会变成：

```text
一个 tool_use 对应两个 tool_result
```

所以完成后用独立通知：

```xml
<task_notification>
  <task_id>bg_0001</task_id>
  <status>completed</status>
  <summary>...</summary>
</task_notification>
```

### s13 记忆句

```text
启动时返回 tool_result，完成 tool_use 配对；
完成后返回 task_notification，通知后台事件结果。
```

## 4. s14 Cron Scheduler

### 解决什么问题

s13 解决了：

```text
现在有个慢操作，放后台跑。
```

s14 解决的是：

```text
未来某个时间，自动触发 Agent 做事。
```

比如：

```text
每天 9 点跑测试
每 5 分钟检查 CI
明天提醒我继续某个任务
```

### Cron 是什么

cron 是一种定时表达式。

s14 使用 5 段式 cron：

```text
分钟 小时 日期 月份 星期
```

例子：

```text
* * * * *        每分钟
0 9 * * *        每天 9:00
*/5 * * * *      每 5 分钟
0 9 * * 1-5      周一到周五 9:00
```

### CronJob 是什么

```python
@dataclass
class CronJob:
    id: str
    cron: str
    prompt: str
    recurring: bool
    durable: bool
```

含义：

- `id`：定时任务 ID。
- `cron`：什么时候触发。
- `prompt`：触发后要交给 Agent 的消息。
- `recurring`：是否重复执行。
- `durable`：是否保存到磁盘。

### durable 是什么

`durable=True` 表示：

```text
任务定义会写入 .scheduled_tasks.json
```

程序重启后可以重新加载。

但它不表示：

```text
Python 程序关闭后任务还能继续自动执行。
```

原因：

```text
真正负责检查时间的是 Python daemon 调度线程。
程序关闭后，线程也停止。
```

所以：

```text
durable = 任务定义跨重启保存
不是 = 程序关闭后还能自动跑
```

### daemon 是什么

`daemon=True` 表示：

```text
这是后台辅助线程。
主程序退出时，它不会阻止退出，会跟着一起结束。
```

### s14 五层核心框架

这是 s14 最重要的结构：

```text
CronJob：
描述任务本身：什么时候触发，触发后要说什么

Scheduler：
只负责看时间，发现到点任务

cron_queue：
把“已经到点”但“还没处理”的任务存起来

Queue processor：
等 Agent 空闲时，把任务交给 Agent

agent_loop：
把任务变成 messages，让模型处理
```

### 为什么不让 scheduler 直接调用模型

因为 Agent 可能正在运行。

如果 `cron_scheduler_loop` 直接调用模型，可能出现：

```text
用户触发的 agent_loop 正在跑
cron 线程又启动一个 agent_loop
两个地方同时修改 messages
两个地方同时调用工具
两个地方同时写文件
```

所以需要：

```text
cron_queue 先存起来
Queue processor 等 Agent 空闲
agent_loop 再消费
```

这是生产者-队列-消费者设计：

```text
Scheduler 生产事件
cron_queue 缓冲事件
Agent 在合适的时候消费事件
```

### s14 记忆句

```text
调度器只生产事件，Agent 在合适的时候消费事件。
```

## 5. s15 Agent Teams

### 解决什么问题

s15 解决的是：

```text
一个 Agent 做大任务时，上下文和注意力不够，需要多个 Agent 协作。
```

s15 引入：

```text
Lead Agent + teammate Agents + MessageBus
```

### spawn 是什么

`spawn` 表示：

```text
创建并启动。
```

在 s15 里：

```text
Lead Agent spawn 一个 teammate
```

意思是：

```text
主 Agent 创建并启动一个队友 Agent。
```

具体代码是：

```python
spawn_teammate_thread(name, role, prompt)
```

然后 Python 开一个后台线程，让队友运行自己的 agent loop。

### s15 和 s06 subagent 的区别

| 对比 | s06 Subagent | s15 Teammate |
|---|---|---|
| 生命周期 | 一次性 | 多轮，教学版最多 10 轮 |
| 通信 | 只返回最终总结 | 通过 mailbox 异步通信 |
| 身份 | 临时 | 有 name / role |
| messages | 子 Agent 临时 messages | 队友自己的 messages |
| 适合场景 | 临时分包 | 多 Agent 协作 |

最短记法：

```text
subagent = 临时外包
teammate = 有名字的队友
```

### MessageBus 是什么

`MessageBus` 是文件消息总线。

每个 Agent 有一个邮箱文件：

```text
.mailboxes/lead.jsonl
.mailboxes/alice.jsonl
.mailboxes/bob.jsonl
```

发消息：

```text
往对方的 .jsonl 文件 append 一行 JSON
```

读消息：

```text
读取自己的 inbox 文件，读完后删除文件
```

这叫消费式邮箱：

```text
消息读一次就被消费掉。
```

### 为什么 teammate 有自己的 messages

因为 teammate 是一个独立 Agent。

它需要自己的：

```text
任务目标
工具调用记录
文件读取结果
bash 输出
收件箱消息
中间上下文
```

如果所有队友共用 Lead 的 `messages`，会导致：

```text
上下文混乱
messages 过长
并行时顺序难控制
职责不清楚
```

所以：

```text
Lead 有 Lead 的 messages
alice 有 alice 的 messages
bob 有 bob 的 messages
```

它们只通过 `MessageBus` 交换必要信息。

### 为什么队友消息要注入 Lead messages

因为模型只能看到 `messages`。

如果 alice 发消息：

```text
schema.sql created
```

但 Python 只是把它存在：

```text
.mailboxes/lead.jsonl
```

Lead 模型并不知道。

所以 Python 必须读取 inbox，然后追加到 Lead 的 `history/messages`：

```python
history.append({
    "role": "user",
    "content": "[Inbox]\nFrom alice: schema.sql created"
})
```

下一次调用模型时，Lead 才能看到队友结果。

### s15 记忆句

```text
各 Agent 保持上下文隔离，只通过 mailbox 共享必要结果。
```

## 6. 易混点总表

| 容易混的概念 | 区别 |
|---|---|
| `max_tokens` vs `prompt_too_long` | 前者是输出不够，后者是输入太长 |
| `429` vs `529` | 429 是你请求太频繁，529 是服务端过载 |
| `todo_write` vs Task System | 前者是当前计划，后者是持久任务图 |
| `blockedBy` vs `owner` | `blockedBy` 是依赖，`owner` 是谁认领 |
| background task vs cron job | background 是现在开始后台跑，cron 是未来到点触发 |
| `tool_result` vs `task_notification` | 前者回复某个 tool_use，后者通知异步事件完成 |
| durable vs daemon | durable 是任务定义保存到磁盘，daemon 是后台线程随主程序退出 |
| subagent vs teammate | subagent 一次性返回总结，teammate 可通过邮箱多轮协作 |
| mailbox vs messages | mailbox 是文件中转站，messages 是模型真正能看到的上下文 |

## 7. 统一心智模型

从 s11 到 s15，其实一直围绕一个核心：

```text
Python 负责维护外部世界和程序状态；
模型只通过 messages 看见这些状态；
所以所有重要事件最终都要进入 messages。
```

具体对应：

```text
s11：
错误恢复后，继续用 messages 调模型

s12：
任务状态写在 .tasks，但模型要通过工具结果看到

s13：
后台完成后，用 task_notification 注入 messages

s14：
cron 到点后，把 prompt 注入 messages

s15：
队友发消息后，把 inbox 注入 Lead messages
```

所以最重要的复习句是：

```text
模型不直接感知文件、线程、时间、邮箱；Python 必须把这些事件转换成 messages。
```

## 8. 二刷题单

1. 为什么第一次遇到 `max_tokens` 不立刻 append 截断输出？
2. 为什么 `429/529` 用 `with_retry`，而 `prompt_too_long` 要压缩 `messages`？
3. `todo_write` 和 s12 Task System 最大区别是什么？
4. `blockedBy` 是什么意思？为什么 `claim_task` 前要 `can_start`？
5. 后台任务启动后为什么必须立刻返回占位 `tool_result`？
6. 后台完成后为什么用 `<task_notification>`，不能再用同一个 `tool_use_id`？
7. `CronJob -> Scheduler -> cron_queue -> Queue processor -> agent_loop` 每层职责是什么？
8. `durable=True` 是否代表 Python 程序关闭后 cron 仍然自动执行？为什么？
9. s06 subagent 和 s15 teammate 有什么区别？
10. 为什么 teammate 要有自己的 `messages`？
11. 为什么队友消息必须注入 Lead 的 `history/messages`？
12. 为什么说“模型只能通过 messages 看见外部事件”？
