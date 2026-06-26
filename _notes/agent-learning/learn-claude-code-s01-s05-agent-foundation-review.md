---
title: "learn-claude-code s01-s05：Agent Foundation 复习笔记"
date: 2026-06-26 16:06:23 +0800
tags: [Agent, Claude Code, Coding Agent, Tool Use, Permission, Hooks, TodoWrite, 源码学习]
main_category: "Agent"
sub_category: "learn-claude-code"
discipline: "Agent"
course: "learn-claude-code"
material_type: "源码学习"
description: "复习 learn-claude-code s01-s05：从最小 agent loop、多工具分发、permission pipeline、hook system 到 todo_write 任务状态管理。"
---

创建日期：2026-06-26  
适用范围：`s01_agent_loop` 到 `s05_todo_write`  
目标：复习最小 coding agent harness 的核心结构、执行原理、代码分层和常见混淆点。

---

## 0. 先背这五句话

1. `s01`：Agent 的最小闭环是 `LLM -> tool_use -> Python 执行工具 -> tool_result -> LLM`。
2. `s02`：新增工具时，不改主循环，只加 `TOOLS` schema、工具函数、`TOOL_HANDLERS` 映射。
3. `s03`：权限必须在 Python 执行层拦截，不能只靠 prompt 让模型自觉。
4. `s04`：hook 是程序预留的生命周期插入点，用来把权限、日志、统计等逻辑移出主循环。
5. `s05`：`todo_write` 是规划和状态工具，让模型显式管理任务，而不是只在“脑子里想”。

---

## 1. 最大心智模型：Model + Harness

这个项目不是在“写一个智能大脑”。智能来自模型本身。代码做的是 harness，也就是给模型提供一个可以行动的环境。

```text
Model 负责：
- 理解用户任务
- 判断下一步该做什么
- 决定要不要调用工具
- 根据 tool_result 继续推理

Harness 负责：
- 告诉模型有哪些工具可用
- 接收模型返回的 tool_use
- 执行真实 Python 函数
- 控制权限和安全边界
- 把工具结果放回 messages
```

所以要一直记住：

```text
模型负责“决定”
Python 负责“执行”
```

模型不会真的自己读文件、写文件、运行 bash。它只是返回一个结构化请求：

```text
我要调用哪个工具？
参数是什么？
```

Python 收到后，才真正执行本地函数。

---

## 2. Agent Loop 总执行流

s01 到 s05 的主循环本质都没变，只是周围的机制越来越多。

```text
用户输入
  |
  v
messages.append({"role": "user", "content": query})
  |
  v
client.messages.create(model, system, messages, tools)
  |
  v
模型返回 response
  |
  +-- stop_reason != "tool_use"
  |      |
  |      v
  |    最终回答，agent_loop 结束
  |
  +-- stop_reason == "tool_use"
         |
         v
      遍历 response.content
         |
         v
      找到 block.type == "tool_use"
         |
         v
      Python 执行对应工具函数
         |
         v
      results.append({"type": "tool_result", ...})
         |
         v
      messages.append({"role": "user", "content": results})
         |
         v
      回到 LLM，继续循环
```

最核心的循环代码在 `s01_agent_loop/code.py:85`：

```python
def agent_loop(messages: list):
    while True:
        response = client.messages.create(...)
        messages.append({"role": "assistant", "content": response.content})

        if response.stop_reason != "tool_use":
            return

        results = []
        for block in response.content:
            if block.type == "tool_use":
                output = run_bash(block.input["command"])
                results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": output,
                })

        messages.append({"role": "user", "content": results})
```

这里最值得反复看的是最后一行：

```python
messages.append({"role": "user", "content": results})
```

工具结果被包装成一条新的 user message，再送回模型。模型看到工具结果后，才知道下一步怎么做。

---

## 3. s01：最小 Agent Loop

文件：`s01_agent_loop/code.py`

### 3.1 s01 增加了什么能力

s01 只有一个工具：`bash`。

模型可以请求 Python 执行一个 shell command。比如模型返回：

```python
block.name = "bash"
block.input = {"command": "ls"}
```

Python 就会执行：

```python
run_bash("ls")
```

### 3.2 TOOLS schema 是给模型看的说明书

代码位置：`s01_agent_loop/code.py:57`

```python
TOOLS = [{
    "name": "bash",
    "description": "Run a shell command.",
    "input_schema": {
        "type": "object",
        "properties": {"command": {"type": "string"}},
        "required": ["command"],
    },
}]
```

它告诉模型三件事：

```text
工具名：bash
作用：运行 shell 命令
参数：必须传 command，而且 command 是 string
```

注意：模型看到的是这个 schema，不是 `run_bash()` 函数本身。

### 3.3 run_bash 是真正执行动作的地方

代码位置：`s01_agent_loop/code.py:69`

```python
def run_bash(command: str) -> str:
    dangerous = ["rm -rf /", "sudo", "shutdown", "reboot", "> /dev/"]
    if any(d in command for d in dangerous):
        return "Error: Dangerous command blocked"
    try:
        r = subprocess.run(command, shell=True, cwd=os.getcwd(),
                           capture_output=True, text=True, timeout=120)
        out = (r.stdout + r.stderr).strip()
        return out[:50000] if out else "(no output)"
    except subprocess.TimeoutExpired:
        return "Error: Timeout (120s)"
    except (FileNotFoundError, OSError) as e:
        return f"Error: {e}"
```

这一段分三层：

```text
第一层：危险命令过滤
第二层：subprocess.run 真正执行 shell command
第三层：try/except 把异常转成字符串返回给模型
```

为什么返回字符串，而不是直接抛异常？

因为 Agent loop 需要继续运行。工具失败也应该变成模型能理解的观察结果：

```text
Error: Timeout (120s)
Error: Dangerous command blocked
```

模型看到错误后，可以换一种方式继续尝试。

### 3.4 s01 的设计价值

s01 证明了一件事：

```text
只要有：
1. messages
2. tools schema
3. client.messages.create()
4. tool_result 回传

就能形成最小 agent loop。
```

这是后面所有章节的地基。

---

## 4. s02：从单工具到多工具

文件：`s02_tool_use/code.py`

### 4.1 s02 增加了什么能力

s01 只有 `bash`。s02 新增四个工具：

```text
read_file  - 读文件
write_file - 写文件
edit_file  - 替换文件中的一段文本
glob       - 按通配符查找文件
```

工具数量变多后，主循环不能再写死：

```python
run_bash(block.input["command"])
```

所以 s02 引入了 `TOOL_HANDLERS`。

### 4.2 safe_path：所有文件操作的安全入口

代码位置：`s02_tool_use/code.py:66`

```python
def safe_path(p: str) -> Path:
    path = (WORKDIR / p).resolve()
    if not path.is_relative_to(WORKDIR):
        raise ValueError(f"Path escapes workspace: {p}")
    return path
```

它解决的问题是：模型可能请求读写工作区外面的路径。

例如：

```text
../../../etc/passwd
/Users/someone/.ssh/id_rsa
```

`safe_path()` 会把路径解析成绝对路径，然后检查它是否仍在 `WORKDIR` 内。

所以文件类工具都先调用它：

```python
safe_path(path).read_text()
safe_path(path).write_text(content)
```

设计原则：

```text
不要相信模型传来的路径。
所有真实文件操作前，都先做边界检查。
```

### 4.3 四个文件工具的职责

代码位置：`s02_tool_use/code.py:73`

```text
run_read(path, limit)
- 读取文件内容
- limit 可以限制读取行数

run_write(path, content)
- 写入完整文件内容
- 如果父目录不存在，会 mkdir

run_edit(path, old_text, new_text)
- 只替换第一次出现的 old_text
- 如果 old_text 不存在，返回错误

run_glob(pattern)
- 根据通配符查找文件
- 只返回路径列表，不读取文件内容
```

`run_edit()` 里的 `replace(old_text, new_text, 1)` 很重要：

```python
text.replace(old_text, new_text, 1)
```

最后的 `1` 表示只替换第一次，避免一次性误改多个地方。

### 4.4 TOOL_HANDLERS：从工具名到函数的路由表

代码位置：`s02_tool_use/code.py:138`

```python
TOOL_HANDLERS = {
    "bash": run_bash,
    "read_file": run_read,
    "write_file": run_write,
    "edit_file": run_edit,
    "glob": run_glob,
}
```

它的作用是：

```text
模型返回工具名
Python 根据工具名找到真实函数
```

主循环变成：

代码位置：`s02_tool_use/code.py:165`

```python
handler = TOOL_HANDLERS.get(block.name)
output = handler(**block.input) if handler else f"Unknown: {block.name}"
```

这里有两个关键点：

1. `block.name` 是模型选择的工具名。
2. `**block.input` 是把字典展开成函数参数。

例如：

```python
block.name = "read_file"
block.input = {"path": "README.md", "limit": 20}
```

实际等价于：

```python
run_read(path="README.md", limit=20)
```

### 4.5 s02 的设计价值

s02 形成了扩展工具的标准方法：

```text
新增一个工具需要三步：

1. 写真实 Python 函数
2. 在 TOOLS 里写 schema 给模型看
3. 在 TOOL_HANDLERS 里把 name 映射到函数
```

主循环不需要理解每个工具的细节，只负责分发。

---

## 5. s03：Permission 权限系统

文件：`s03_permission/code.py`

### 5.1 s03 解决的问题

s02 给模型提供了读写文件和运行命令的能力，但能力越大，风险越大。

模型可能请求：

```text
rm file
chmod 777
写入工作区外的路径
sudo ...
```

所以 s03 在工具真正执行前，加了 permission pipeline。

核心原则：

```text
安全不能只靠 system prompt。
安全必须在 Python 执行前拦截。
```

### 5.2 三道权限门

代码位置：`s03_permission/code.py:148`

第一道：硬禁止列表。

```python
DENY_LIST = ["rm -rf /", "sudo", "shutdown", "reboot", "mkfs", "dd if=", "> /dev/sda"]
```

这些命令一旦匹配，不问用户，直接拒绝。

第二道：规则检查。

代码位置：`s03_permission/code.py:159`

```python
PERMISSION_RULES = [
    {"tools": ["write_file", "edit_file"],
     "check": lambda args: not (WORKDIR / args.get("path", "")).resolve().is_relative_to(WORKDIR),
     "message": "Writing outside workspace"},
    {"tools": ["bash"],
     "check": lambda args: any(kw in args.get("command", "") for kw in ["rm ", "> /etc/", "chmod 777"]),
     "message": "Potentially destructive command"},
]
```

这些规则不是永远禁止，而是需要用户确认。

第三道：用户审批。

代码位置：`s03_permission/code.py:176`

```python
choice = input("   Allow? [y/N] ").strip().lower()
return "allow" if choice in ("y", "yes") else "deny"
```

### 5.3 check_permission 是权限入口

代码位置：`s03_permission/code.py:184`

```python
def check_permission(block) -> bool:
    if block.name == "bash":
        reason = check_deny_list(block.input.get("command", ""))
        if reason:
            print(...)
            return False

    reason = check_rules(block.name, block.input)
    if reason:
        decision = ask_user(block.name, block.input, reason)
        if decision == "deny":
            return False

    return True
```

它接收的是 `block`，也就是模型返回的 tool_use。

为什么传 `block` 而不是只传 command？

因为不同工具需要不同检查：

```text
bash      -> 看 command
write_file -> 看 path
edit_file  -> 看 path、old_text、new_text
```

### 5.4 权限插入点

代码位置：`s03_permission/code.py:220`

```python
if not check_permission(block):
    results.append({
        "type": "tool_result",
        "tool_use_id": block.id,
        "content": "Permission denied."
    })
    continue
```

含义：

```text
工具执行前先检查权限
如果不允许：
  不执行 handler
  给模型返回 Permission denied
  继续处理下一个工具调用
```

注意：即使拒绝工具，也要返回 `tool_result`。因为模型已经发出了一个 tool_use，协议上需要给它一个对应结果。

### 5.5 s03 的设计价值

s03 把 Agent 的行动能力放进安全边界里。

这是一条非常重要的工程原则：

```text
模型可以提出动作请求。
Python 必须决定这个动作能不能真的发生。
```

---

## 6. s04：Hooks 扩展系统

文件：`s04_hooks/code.py`

### 6.1 s04 解决的问题

s03 已经有权限检查，但权限逻辑是直接写进 `agent_loop` 的。

如果以后还想加：

```text
记录每次工具调用
统计用了多少工具
检查输出是否过大
在用户输入前补充上下文
工具执行后自动做清理
```

都往 `agent_loop` 里塞，主循环会越来越乱。

s04 的方法是：

```text
主循环只保留几个“触发点”
具体功能函数注册到这些触发点上
```

这就是 hook。

### 6.2 hook 的本质

代码位置：`s04_hooks/code.py:159`

```python
HOOKS = {"UserPromptSubmit": [], "PreToolUse": [], "PostToolUse": [], "Stop": []}
```

这就是一个普通字典：

```text
key   = 事件名，也就是时机
value = 函数列表，也就是这个时机要执行的 callback
```

可以理解成：

```text
UserPromptSubmit 抽屉：用户提交输入时要运行的函数
PreToolUse 抽屉：工具执行前要运行的函数
PostToolUse 抽屉：工具执行后要运行的函数
Stop 抽屉：Agent 停止前要运行的函数
```

### 6.3 register_hook：把函数放进抽屉

代码位置：`s04_hooks/code.py:161`

```python
def register_hook(event: str, callback):
    HOOKS[event].append(callback)
```

注册时不会执行函数。它只是保存函数对象。

例如：

代码位置：`s04_hooks/code.py:225`

```python
register_hook("PreToolUse", permission_hook)
register_hook("PreToolUse", log_hook)
```

意思是：

```text
以后每次触发 PreToolUse，
依次执行 permission_hook 和 log_hook。
```

### 6.4 trigger_hooks：到了时机才真正执行

代码位置：`s04_hooks/code.py:164`

```python
def trigger_hooks(event: str, *args):
    for callback in HOOKS[event]:
        result = callback(*args)
        if result is not None:
            return result
    return None
```

这段代码很关键。它实际做的是：

```text
取出 HOOKS[event] 里的函数列表
一个一个调用
如果某个 hook 返回了非 None，立刻返回
如果所有 hook 都返回 None，就说明没有拦截
```

如果现在：

```python
HOOKS["PreToolUse"] = [permission_hook, log_hook]
```

那么：

```python
trigger_hooks("PreToolUse", block)
```

等价于：

```python
result = permission_hook(block)
if result is not None:
    return result

result = log_hook(block)
if result is not None:
    return result

return None
```

所以 hook 不神秘。它就是：

```text
字典 + 函数列表 + for 循环调用
```

### 6.5 四个 hook 时机

代码里有四类事件：

```text
UserPromptSubmit
- 用户输入后，真正加入 messages 前触发
- 当前用于打印工作目录

PreToolUse
- 工具执行前触发
- 当前用于权限检查和日志记录

PostToolUse
- 工具执行后触发
- 当前用于检查大输出

Stop
- 模型不再调用工具、agent_loop 准备退出时触发
- 当前用于打印工具调用统计
```

触发位置：

```python
trigger_hooks("UserPromptSubmit", query)       # s04_hooks/code.py:287
blocked = trigger_hooks("PreToolUse", block)  # s04_hooks/code.py:259
trigger_hooks("PostToolUse", block, output)   # s04_hooks/code.py:268
force = trigger_hooks("Stop", messages)       # s04_hooks/code.py:247
```

### 6.6 hook 和 tool 的区别

这是最容易混的地方。

```text
tool：
- 给模型看的能力
- 模型可以主动请求调用
- 有 schema
- 通过 TOOL_HANDLERS 执行
- 例子：bash, read_file, write_file, edit_file, glob

hook：
- Python 程序内部的生命周期扩展点
- 模型通常不知道
- 没有 tool schema
- 由 trigger_hooks 在固定时机调用
- 例子：permission_hook, log_hook, summary_hook
```

一句话：

```text
tool 是 Agent 的“手”，负责做事。
hook 是流程里的“检查点”，负责在做事前后插入逻辑。
```

### 6.7 s04 的设计价值

s04 让主循环保持稳定：

```python
blocked = trigger_hooks("PreToolUse", block)
...
trigger_hooks("PostToolUse", block, output)
```

以后新增一个功能，不需要继续改 `agent_loop`，只要：

```python
def new_hook(...):
    ...

register_hook("某个事件", new_hook)
```

这就是扩展点的价值。

---

## 7. s05：TodoWrite 规划工具

文件：`s05_todo_write/code.py`

### 7.1 s05 解决的问题

前面 Agent 已经能调用工具、检查权限、走 hook。

但还有一个问题：复杂任务如果没有显式计划，模型容易边做边忘，或者做完一部分就偏离目标。

s05 引入 `todo_write`，让模型在多步骤任务前先写任务列表，并在执行过程中更新状态。

### 7.2 SYSTEM prompt 提醒模型先规划

代码位置：`s05_todo_write/code.py:53`

```python
SYSTEM = (
    f"You are a coding agent at {WORKDIR}. "
    "Before starting any multi-step task, use todo_write to plan your steps. "
    "Update status as you go."
)
```

它不是强制规则，而是行为引导：

```text
多步骤任务前，先用 todo_write 规划。
执行过程中，更新任务状态。
```

真正让模型能调用 `todo_write` 的，是后面的 tool schema。

### 7.3 CURRENT_TODOS：内存里的任务状态

代码位置：`s05_todo_write/code.py:50`

```python
CURRENT_TODOS: list[dict] = []
```

这表示当前任务列表只存在程序内存里。

特点：

```text
程序运行期间有效
程序退出后消失
不是写入文件
不是数据库
```

### 7.4 _normalize_todos：校验模型传来的数据

代码位置：`s05_todo_write/code.py:124`

```python
def _normalize_todos(todos):
    if isinstance(todos, str):
        try:
            todos = json.loads(todos)
        except json.JSONDecodeError:
            try:
                todos = ast.literal_eval(todos)
            except (SyntaxError, ValueError):
                return None, "Error: todos must be a list or JSON array string"
    if not isinstance(todos, list):
        return None, "Error: todos must be a list"
    ...
    return todos, None
```

它做的是输入清洗和格式校验。

为什么需要它？

虽然 schema 告诉模型 `todos` 应该是数组，但模型输出仍可能不完美。执行层最好再检查一次。

它接受两种形式：

```text
1. 真实 list
2. JSON 字符串或 Python 字面量字符串
```

每个 todo 必须是：

```python
{
    "content": "...",
    "status": "pending" | "in_progress" | "completed"
}
```

函数名前面的 `_` 是约定：这是内部辅助函数，不是外部主要接口。

### 7.5 run_todo_write：保存并展示任务

代码位置：`s05_todo_write/code.py:144`

```python
def run_todo_write(todos: list) -> str:
    global CURRENT_TODOS
    todos, error = _normalize_todos(todos)
    if error:
        return error
    CURRENT_TODOS = todos
    ...
    return f"Updated {len(CURRENT_TODOS)} tasks"
```

这里做三件事：

```text
1. 校验 todos
2. 更新全局变量 CURRENT_TODOS
3. 打印当前任务列表，并返回更新结果
```

注意：`todo_write` 不执行任务。

它只是记录计划：

```text
pending      还没开始
in_progress 进行中
completed   已完成
```

### 7.6 todo_write schema

代码位置：`s05_todo_write/code.py:169`

```python
{"name": "todo_write",
 "description": "Create and manage a task list for your current coding session.",
 "input_schema": {
     "type": "object",
     "properties": {
         "todos": {
             "type": "array",
             "items": {
                 "type": "object",
                 "properties": {
                     "content": {"type": "string"},
                     "status": {"type": "string", "enum": ["pending", "in_progress", "completed"]}
                 },
                 "required": ["content", "status"]
             }
         }
     },
     "required": ["todos"]
 }}
```

schema 让模型知道：

```text
todo_write 需要一个 todos 参数
todos 是数组
数组里每项都有 content 和 status
status 只能是三个值之一
```

### 7.7 reminder 机制

代码位置：`s05_todo_write/code.py:235`

```python
rounds_since_todo = 0
```

代码位置：`s05_todo_write/code.py:241`

```python
if rounds_since_todo >= 3 and messages:
    messages.append({"role": "user",
                     "content": "<reminder>Update your todos.</reminder>"})
    rounds_since_todo = 0
```

如果模型连续几轮调用工具但没有更新 todo，程序会向 messages 里插入一条提醒：

```text
<reminder>Update your todos.</reminder>
```

当模型调用 `todo_write` 时，计数归零：

代码位置：`s05_todo_write/code.py:276`

```python
if block.name == "todo_write":
    rounds_since_todo = 0
```

### 7.8 s05 的设计价值

s05 让 Agent 不只是“会做事”，还开始“管理任务状态”。

```text
s01-s04：让 Agent 能行动，并且安全、可扩展。
s05：让 Agent 的行动变得有计划。
```

---

## 8. 五节之间的演进关系

| 章节 | 新增内容 | 核心变化 | 主循环是否大改 |
| --- | --- | --- | --- |
| s01 | `bash` | 最小 agent loop | 是，建立基础 |
| s02 | 多工具 + `TOOL_HANDLERS` | 从硬编码工具变成通用分发 | 小改工具执行部分 |
| s03 | permission pipeline | 执行前加安全门 | 插入权限检查 |
| s04 | hook system | 把扩展逻辑移出主循环 | 用 trigger 代替硬编码检查 |
| s05 | `todo_write` | 增加显式计划和状态管理 | 只加 reminder 计数 |

可以用这条线复习：

```text
会行动 -> 行动更多样 -> 行动前检查 -> 检查逻辑可扩展 -> 行动有计划
```

---

## 9. 四个最容易混的概念

### 9.1 schema vs Python 函数

```text
schema：
- 给模型看的说明
- 说清楚工具名、描述、参数结构
- 不执行任何东西

Python 函数：
- 给程序执行的真实动作
- 比如 run_bash、run_read、run_todo_write
```

模型只看到 schema。Python 通过 tool name 找到函数。

### 9.2 tool_use vs tool_result

```text
tool_use：
- 模型发出的请求
- 意思是：我想调用某个工具

tool_result：
- Python 执行工具后的结果
- 意思是：工具已经运行，结果如下
```

流程：

```text
模型 -> tool_use
Python -> tool_result
模型 -> 根据 tool_result 继续
```

### 9.3 tool vs hook

```text
tool：
- 模型主动调用
- 例如 bash/read_file/todo_write
- 是行动能力

hook：
- 程序在固定时机自动调用
- 例如 permission_hook/log_hook
- 是流程扩展能力
```

### 9.4 permission vs safe_path

```text
permission：
- 工具执行前的策略层检查
- 可以询问用户是否允许
- s03 开始出现

safe_path：
- 文件操作内部的硬边界
- 防止路径逃出工作区
- s02 开始出现
```

两者可以同时存在：

```text
permission 先决定“允不允许尝试”
safe_path 再保证“文件路径不能越界”
```

---

## 10. 读 Agent 代码的方法

以后看到新的 Agent 代码，可以按这 7 个问题读：

1. 入口在哪里？用户输入怎么进入 `messages`？
2. `client.messages.create()` 传了哪些参数？
3. `TOOLS` 暴露了哪些工具给模型？
4. 每个工具有没有对应的 Python handler？
5. `tool_use` 是在哪里被遍历和执行的？
6. 执行前有没有 permission/hook/sandbox？
7. `tool_result` 是如何回到 `messages` 的？

这 7 个问题能帮你快速看懂大多数 tool-use agent。

---

## 11. 口述版总结

如果别人问你：这个项目 s01-s05 在讲什么？

可以这样回答：

> 这五节是在搭一个最小 coding agent harness。s01 先实现最核心的 agent loop：模型返回 tool_use，Python 执行工具，再把 tool_result 喂回模型。s02 把单一 bash 工具扩展成多工具系统，并用 TOOL_HANDLERS 做分发。s03 在工具执行前加入 permission pipeline，保证危险操作不会直接执行。s04 把权限、日志、统计等扩展逻辑抽成 hook，让主循环保持干净。s05 增加 todo_write，让模型在复杂任务中显式规划和更新状态。

更短版本：

> 模型负责决策，Python harness 负责执行工具、控制权限、管理上下文和状态。

---

## 12. 自测题

### 题 1

模型能不能直接调用 `run_bash()` 这个 Python 函数？

答案：不能。模型只能看到 `TOOLS` schema。它返回 `tool_use`，Python 根据 `block.name` 和 `block.input` 决定是否调用 `run_bash()`。

### 题 2

为什么 s02 要加 `TOOL_HANDLERS`？

答案：因为工具变多后，主循环不能写死 `run_bash()`。`TOOL_HANDLERS` 把工具名映射到真实函数，让主循环可以统一分发。

### 题 3

如果模型请求了危险命令，为什么不能只靠 system prompt 阻止？

答案：prompt 是行为引导，不是安全边界。真正的安全边界必须在 Python 执行前检查，比如 `check_permission()` 或 `permission_hook()`。

### 题 4

hook 是不是 tool？

答案：不是。tool 是模型可以请求的动作；hook 是 Python 程序在固定流程时机自动调用的辅助函数。

### 题 5

`todo_write` 会不会自动执行任务？

答案：不会。它只记录和展示任务计划，真正执行仍然靠 bash/read/write/edit 等工具。

### 题 6

为什么被拒绝的工具调用也要返回 `tool_result`？

答案：因为模型已经发出了一个 tool_use，协议上需要给它对应结果。即使结果是 `"Permission denied."`，也要让模型知道这个工具调用的结局。

### 题 7

`**block.input` 是什么意思？

答案：把字典展开成函数参数。例如 `{"path": "a.txt", "limit": 10}` 会变成 `run_read(path="a.txt", limit=10)`。

---

## 13. 代码定位索引

### s01

- `s01_agent_loop/code.py:57` - `bash` tool schema
- `s01_agent_loop/code.py:69` - `run_bash()`
- `s01_agent_loop/code.py:85` - 最小 `agent_loop()`
- `s01_agent_loop/code.py:87` - `client.messages.create()`
- `s01_agent_loop/code.py:106` - 构造 `tool_result`
- `s01_agent_loop/code.py:113` - 工具结果回到 `messages`

### s02

- `s02_tool_use/code.py:66` - `safe_path()`
- `s02_tool_use/code.py:73` - `run_read()`
- `s02_tool_use/code.py:83` - `run_write()`
- `s02_tool_use/code.py:93` - `run_edit()`
- `s02_tool_use/code.py:105` - `run_glob()`
- `s02_tool_use/code.py:121` - 多工具 `TOOLS`
- `s02_tool_use/code.py:138` - `TOOL_HANDLERS`
- `s02_tool_use/code.py:165` - 通用工具分发

### s03

- `s03_permission/code.py:148` - `DENY_LIST`
- `s03_permission/code.py:159` - `PERMISSION_RULES`
- `s03_permission/code.py:176` - `ask_user()`
- `s03_permission/code.py:184` - `check_permission()`
- `s03_permission/code.py:220` - 执行工具前的权限拦截点

### s04

- `s04_hooks/code.py:159` - `HOOKS`
- `s04_hooks/code.py:161` - `register_hook()`
- `s04_hooks/code.py:164` - `trigger_hooks()`
- `s04_hooks/code.py:176` - `permission_hook()`
- `s04_hooks/code.py:200` - `log_hook()`
- `s04_hooks/code.py:206` - `large_output_hook()`
- `s04_hooks/code.py:218` - `summary_hook()`
- `s04_hooks/code.py:225` - hook 注册
- `s04_hooks/code.py:259` - `PreToolUse` 触发
- `s04_hooks/code.py:268` - `PostToolUse` 触发
- `s04_hooks/code.py:287` - `UserPromptSubmit` 触发

### s05

- `s05_todo_write/code.py:50` - `CURRENT_TODOS`
- `s05_todo_write/code.py:53` - 提醒模型先规划的 `SYSTEM`
- `s05_todo_write/code.py:124` - `_normalize_todos()`
- `s05_todo_write/code.py:144` - `run_todo_write()`
- `s05_todo_write/code.py:169` - `todo_write` schema
- `s05_todo_write/code.py:173` - `todo_write` 加入 `TOOL_HANDLERS`
- `s05_todo_write/code.py:235` - `rounds_since_todo`
- `s05_todo_write/code.py:241` - reminder 注入
- `s05_todo_write/code.py:276` - 调用 `todo_write` 后重置计数

---

## 14. 推荐复习顺序

第一次复习：

```text
1. 读第 2 节 Agent Loop 总执行流
2. 读第 9 节 四个易混概念
3. 读 s01 和 s02
4. 读 s03 和 s04
5. 读 s05
6. 做第 12 节自测题
```

第二次复习：

```text
1. 只看第 13 节代码索引
2. 跳到对应源码位置
3. 自己口述每一处代码在流程里的作用
```

第三次复习：

```text
尝试不用看笔记，画出这条线：

用户输入
-> messages
-> client.messages.create
-> response.content
-> tool_use block
-> permission/hook
-> TOOL_HANDLERS
-> tool_result
-> messages
-> loop
```

能画出来，s01-s05 的骨架就掌握了。
