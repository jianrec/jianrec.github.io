---
title: "learn-claude-code s06-s10：Context And Capability 复习笔记"
date: 2026-06-27 20:31:43 +0800
tags: [Agent, Claude Code, Coding Agent, Subagent, Skill Loading, Context Compact, Memory, System Prompt]
main_category: "Agent"
sub_category: "learn-claude-code"
discipline: "Agent"
course: "learn-claude-code"
material_type: "源码学习"
description: "复习 learn-claude-code s06-s10：subagent 上下文隔离、skill 按需加载、context compact、memory 长期记忆和动态 system prompt。"
---

创建日期：2026-06-26  
适用范围：`s06_subagent` 到 `s10_system_prompt`  
目标：复习 Agent harness 在上下文隔离、知识加载、上下文压缩、长期记忆、system prompt 组装上的演进。

---

## 0. 先背这五句话

1. `s06`：Subagent 用新的 `messages[]` 单独完成子任务，主 Agent 只拿总结。
2. `s07`：Skill Loading 先给模型技能目录，需要时再用 `load_skill` 加载完整 `SKILL.md`。
3. `s08`：Context Compact 在每次 LLM 调用前清理 `messages`，避免上下文爆掉。
4. `s09`：Memory 把长期有用的信息写到 `.memory/`，跨压缩、跨会话保存。
5. `s10`：System Prompt 不再写死，而是按当前 `context` 运行时组装。

---

## 1. s06-s10 的总主线

`s01-s05` 解决的是 Agent 能不能行动：

```text
tool_use -> Python handler -> tool_result -> loop
```

`s06-s10` 解决的是 Agent 怎么长期、专业、稳定地行动：

```text
s06: 复杂过程太吵 -> 用 subagent 隔离
s07: 专业知识太多 -> 用 skill 按需加载
s08: 历史上下文太长 -> 用 compact 压缩
s09: 压缩会丢重要信息 -> 用 memory 长期保存
s10: system prompt 太乱 -> 用 section 运行时组装
```

可以把这五节看成同一个问题的不同侧面：

```text
上下文是有限资源。

subagent: 不让子任务过程污染主上下文
skill: 不让无关知识提前进入上下文
compact: 不让历史消息无限增长
memory: 不让重要信息因为压缩而消失
system prompt: 不让系统指令变成一坨不可维护文本
```

---

## 2. s06：Subagent，上下文隔离

文件：`s06_subagent/code.py`

### 2.1 s06 解决的问题

复杂任务会产生大量中间过程。比如：

```text
读 20 个文件
跑 10 条命令
分析调用链
最后只需要一句结论
```

如果这些中间过程全留在主 Agent 的 `messages`，主上下文会变得很吵。  
所以 s06 加了 `task` 工具，让主 Agent 可以派一个子 Agent 去做支线任务。

### 2.2 task 是普通 tool

代码位置：`s06_subagent/code.py:252`

```python
TOOLS.append({
    "name": "task",
    "description": "Launch a subagent to handle a complex subtask. Returns only the final conclusion.",
    "input_schema": {"type": "object", "properties": {"description": {"type": "string"}}, "required": ["description"]},
})
TOOL_HANDLERS["task"] = spawn_subagent
```

这里和之前工具机制完全一样：

```text
模型看到 task schema
-> 模型返回 tool_use: task
-> Python 查 TOOL_HANDLERS["task"]
-> 执行 spawn_subagent(description)
```

所以 `task` 没有魔法，它只是一个 handler 比较特殊的 tool。

### 2.3 子 Agent 有自己的 system prompt

代码位置：`s06_subagent/code.py:58`

```python
SUB_SYSTEM = (
    f"You are a coding agent at {WORKDIR}. "
    "Complete the task you were given, then return a concise summary. "
    "Do not delegate further."
)
```

这说明子 Agent 的目标更窄：

```text
完成交给你的任务
返回简短总结
不要继续委派
```

### 2.4 子 Agent 有自己的工具集合

代码位置：`s06_subagent/code.py:182`

```python
SUB_TOOLS = [
    bash,
    read_file,
    write_file,
    edit_file,
    glob,
]
```

注意：`SUB_TOOLS` 没有 `task`。

目的：

```text
防止主 Agent -> 子 Agent -> 子子 Agent -> 无限递归
```

### 2.5 spawn_subagent 是核心

代码位置：`s06_subagent/code.py:207`

```python
def spawn_subagent(description: str) -> str:
    print("[Subagent spawned]")
    messages = [{"role": "user", "content": description}]
```

最关键是这一句：

```python
messages = [{"role": "user", "content": description}]
```

它创建的是子 Agent 自己的局部 `messages`，不是主 Agent 的 `history`。

所以：

```text
主 Agent messages: 用户原始任务 + 主流程历史
子 Agent messages: 子任务 description + 子 Agent 内部过程
```

这就是 context isolation。

### 2.6 子 Agent 自己跑 loop

代码位置：`s06_subagent/code.py:212`

```python
for _ in range(30):
    response = client.messages.create(
        model=MODEL,
        system=SUB_SYSTEM,
        messages=messages,
        tools=SUB_TOOLS,
        max_tokens=8000,
    )
```

子 Agent 本质上也有一个小版 agent loop：

```text
调用 LLM
-> 如果 tool_use
-> 执行 SUB_HANDLERS
-> tool_result 放回子 messages
-> 继续
```

`for _ in range(30)` 是安全上限，避免子 Agent 无限循环。

### 2.7 只返回总结，不返回过程

代码位置：`s06_subagent/code.py:249`

```python
return result
```

主 Agent 最后只拿到一个字符串：

```text
子 Agent 的最终总结
```

不会拿到子 Agent 内部的 10 次读文件、5 次 bash 的详细过程。

但要注意：

```text
子 Agent 的 messages 不进入主 Agent。
子 Agent 的真实副作用会保留。
```

例如子 Agent 写了文件，这个文件真的会存在。

### 2.8 s06 复习口诀

```text
task 是入口
spawn_subagent 是执行
fresh messages 是隔离
SUB_TOOLS 无 task 是防递归
return summary 是减上下文
```

---

## 3. s07：Skill Loading，知识按需加载

文件：`s07_skill_loading/code.py`

### 3.1 s07 解决的问题

Agent 需要专业知识，比如：

```text
code review 规范
PDF 处理流程
MCP server 编写方法
agent 架构指南
```

如果一开始全塞进 `SYSTEM`，会浪费 token，也会让上下文很吵。

s07 的设计是两层：

```text
第一层：技能目录常驻 system prompt
第二层：完整 SKILL.md 按需 load_skill 加载
```

### 3.2 skills/ 目录

代码位置：`s07_skill_loading/code.py:47`

```python
SKILLS_DIR = WORKDIR / "skills"
```

项目里的技能文件结构：

```text
skills/
  agent-builder/SKILL.md
  code-review/SKILL.md
  mcp-builder/SKILL.md
  pdf/SKILL.md
```

每个 `SKILL.md` 顶部有 frontmatter：

```yaml
---
name: code-review
description: Perform thorough code reviews...
---
```

### 3.3 _parse_frontmatter 解析短信息

代码位置：`s07_skill_loading/code.py:53`

```python
def _parse_frontmatter(text: str) -> tuple[dict, str]:
    if not text.startswith("---"):
        return {}, text
    parts = text.split("---", 2)
    meta = yaml.safe_load(parts[1]) or {}
    return meta, parts[2].strip()
```

它从 `SKILL.md` 顶部提取：

```text
name
description
```

这一步还没有给 LLM，只是在 Python 内部解析。

### 3.4 _scan_skills 建立注册表

代码位置：`s07_skill_loading/code.py:69`

```python
def _scan_skills():
    ...
    raw = manifest.read_text()
    meta, body = _parse_frontmatter(raw)
    name = meta.get("name", d.name)
    desc = meta.get("description", ...)
    SKILL_REGISTRY[name] = {"name": name, "description": desc, "content": raw}
```

`SKILL_REGISTRY` 是 Python 内存中的技能数据库。

大概长这样：

```python
{
    "code-review": {
        "name": "code-review",
        "description": "Perform thorough code reviews...",
        "content": "完整 SKILL.md 内容"
    }
}
```

注意：LLM 不能直接读 Python 变量。必须转成文本，通过 `system` / `messages` / `tool_result` 发给它。

### 3.5 list_skills 生成轻量目录

代码位置：`s07_skill_loading/code.py:86`

```python
def list_skills() -> str:
    if not SKILL_REGISTRY:
        return "(no skills found)"
    return "\n".join(f"- **{s['name']}**: {s['description']}" for s in SKILL_REGISTRY.values())
```

这一步把 Python 字典转成普通文本菜单：

```text
- **agent-builder**: Design and build AI agents...
- **code-review**: Perform thorough code reviews...
- **mcp-builder**: Build MCP servers...
- **pdf**: Process PDF files...
```

这段文本人类能读，LLM 也能读。

### 3.6 build_system 把目录发给 LLM

代码位置：`s07_skill_loading/code.py:93`

```python
def build_system() -> str:
    catalog = list_skills()
    return (
        f"You are a coding agent at {WORKDIR}. "
        f"Skills available:\n{catalog}\n"
        "Use load_skill to get full details when needed."
    )
```

最终通过：

```python
client.messages.create(..., system=SYSTEM, ...)
```

发给 LLM。

完整链路：

```text
SKILL.md 文件
-> _parse_frontmatter()
-> _scan_skills()
-> SKILL_REGISTRY
-> list_skills()
-> build_system()
-> client.messages.create(system=SYSTEM)
-> LLM
```

### 3.7 load_skill 加载完整内容

代码位置：`s07_skill_loading/code.py:269`

```python
def load_skill(name: str) -> str:
    skill = SKILL_REGISTRY.get(name)
    if not skill:
        return f"Skill not found: {name}"
    return skill["content"]
```

模型看到目录后，如果判断需要某个技能，可以调用：

```python
load_skill({"name": "code-review"})
```

Python 返回完整 `SKILL.md`，作为 `tool_result` 进入 `messages`。

### 3.8 s07 复习口诀

```text
技能文件在 skills/
frontmatter 提取 name/description
SKILL_REGISTRY 存 Python 内存
list_skills 变文本菜单
build_system 发目录
load_skill 发全文
```

---

## 4. s08：Context Compact，上下文压缩

文件：`s08_context_compact/code.py`

### 4.1 s08 解决的问题

`messages` 会持续增长：

```text
user prompt
assistant tool_use
user tool_result
assistant tool_use
user tool_result
...
```

读文件、跑命令、加载 skill 都会增加上下文。  
如果不压缩，最终 API 会报：

```text
prompt_too_long
too many tokens
```

s08 的原则：

```text
便宜的压缩先跑，贵的压缩后跑。
```

### 4.2 三个常量

代码位置：`s08_context_compact/code.py:265`

```python
CONTEXT_LIMIT = 50000
KEEP_RECENT = 3
PERSIST_THRESHOLD = 30000
```

含义：

```text
CONTEXT_LIMIT: 超过这个估算大小就触发 LLM 摘要
KEEP_RECENT: 最近 3 个 tool_result 保留完整
PERSIST_THRESHOLD: 单个大输出超过这个长度就落盘
```

教学版用 `len(str(messages))` 估算大小，不是真正 token 数。

### 4.3 snip_compact：裁中间消息

代码位置：`s08_context_compact/code.py:295`

```python
def snip_compact(messages, max_messages=50):
    if len(messages) <= max_messages:
        return messages
    keep_head, keep_tail = 3, max_messages - 3
    ...
    return messages[:head_end] + [{"role": "user", "content": f"[snipped {snipped} messages]"}] + messages[tail_start:]
```

作用：

```text
保留开头几条
保留最近几十条
中间旧消息用占位符替代
```

为什么不能随便切？

因为 `assistant(tool_use)` 和 `user(tool_result)` 是一组，不能留下半组。

### 4.4 micro_compact：旧工具结果占位

代码位置：`s08_context_compact/code.py:322`

```python
def micro_compact(messages):
    tool_results = collect_tool_results(messages)
    if len(tool_results) <= KEEP_RECENT:
        return messages
    for _, _, block in tool_results[:-KEEP_RECENT]:
        if len(block.get("content", "")) > 120:
            block["content"] = "[Earlier tool result compacted. Re-run if needed.]"
    return messages
```

作用：

```text
最近 3 个 tool_result 保留完整
更早的长 tool_result 替换为一句占位符
```

这是有损压缩。旧内容没了，但模型可以重新读文件或重新运行命令。

### 4.5 tool_result_budget：大输出落盘

代码位置：`s08_context_compact/code.py:332`

```python
def persist_large_output(tool_use_id, output):
    if len(output) <= PERSIST_THRESHOLD:
        return output
    TOOL_RESULTS_DIR.mkdir(parents=True, exist_ok=True)
    path = TOOL_RESULTS_DIR / f"{tool_use_id}.txt"
    if not path.exists():
        path.write_text(output)
    return f"<persisted-output>\nFull output: {path}\nPreview:\n{output[:2000]}\n</persisted-output>"
```

作用：

```text
完整大输出写到磁盘
messages 里只保留路径 + 预览
```

这比直接丢掉更稳，因为完整内容还在：

```text
.task_outputs/tool-results/<tool_use_id>.txt
```

### 4.6 compact_history：LLM 摘要

代码位置：`s08_context_compact/code.py:375`

```python
def compact_history(messages):
    transcript_path = write_transcript(messages)
    print(f"[transcript saved: {transcript_path}]")
    summary = summarize_history(messages)
    return [{"role": "user", "content": f"[Compacted]\n\n{summary}"}]
```

作用：

```text
完整历史写 transcript
让 LLM 总结当前目标、关键发现、改动文件、剩余工作、用户约束
用一条摘要替换旧 messages
```

这是最贵的一层，因为需要额外一次 LLM 调用。

### 4.7 reactive_compact：应急压缩

代码位置：`s08_context_compact/code.py:383`

```python
def reactive_compact(messages):
    ...
    return [{"role": "user", "content": f"[Reactive compact]\n\n{summary}"}, *messages[tail_start:]]
```

如果请求 API 时仍然报 `prompt_too_long`，就应急压缩并重试一次。

### 4.8 压缩接入 agent_loop

代码位置：`s08_context_compact/code.py:454`

```python
def agent_loop(messages: list):
    while True:
        messages[:] = tool_result_budget(messages)
        messages[:] = snip_compact(messages)
        messages[:] = micro_compact(messages)

        if estimate_size(messages) > CONTEXT_LIMIT:
            messages[:] = compact_history(messages)

        response = client.messages.create(...)
```

注意实际执行顺序：

```text
tool_result_budget -> snip_compact -> micro_compact -> compact_history
```

虽然讲解里叫 L1/L2/L3/L4，但执行时先落盘大输出，避免后续占位时丢掉完整内容。

### 4.9 compact 工具

代码位置：`s08_context_compact/code.py:416`

```python
{"name": "compact", "description": "Summarize earlier conversation to free context space."}
```

模型可以主动调用 `compact`，触发 `compact_history()`。

### 4.10 s08 复习口诀

```text
messages 越跑越长
大输出先落盘
中间旧消息裁掉
旧 tool_result 占位
还太大就 LLM 摘要
API 报错就 reactive compact
```

---

## 5. s09：Memory，长期记忆

文件：`s09_memory/code.py`

### 5.1 s09 解决的问题

s08 压缩能让会话继续，但压缩是有损的。  
重要信息可能被摘要得太粗，或者新会话直接丢失。

比如用户说：

```text
以后解释代码请用中文，并且多举例子。
```

这不应该只留在当前 `messages`，应该长期保存。

s09 的方案：

```text
把重要信息写到 .memory/ 文件系统
用 MEMORY.md 做索引
每轮开始加载相关记忆
每轮结束提取新记忆
记忆多了再合并整理
```

### 5.2 记忆目录

代码位置：`s09_memory/code.py:42`

```python
MEMORY_DIR = WORKDIR / ".memory"
MEMORY_INDEX = MEMORY_DIR / "MEMORY.md"
```

目录结构：

```text
.memory/
  MEMORY.md
  user-prefers-chinese.md
  project-facts.md
  feedback-style.md
```

### 5.3 四类记忆

代码位置：`s09_memory/code.py:56`

```python
MEMORY_TYPES = ["user", "feedback", "project", "reference"]
```

含义：

```text
user: 用户是谁、用户偏好
feedback: 用户对 Agent 做事方式的反馈
project: 项目事实
reference: 外部线索或参考位置
```

### 5.4 write_memory_file：写单条记忆

代码位置：`s09_memory/code.py:72`

```python
def write_memory_file(name: str, mem_type: str, description: str, body: str):
    slug = name.lower().replace(" ", "-").replace("/", "-")
    filename = f"{slug}.md"
    filepath = MEMORY_DIR / filename
    filepath.write_text(
        f"---\nname: {name}\ndescription: {description}\ntype: {mem_type}\n---\n\n{body}\n"
    )
    _rebuild_index()
    return filepath
```

单条记忆文件格式：

```markdown
---
name: user-prefers-chinese
description: User prefers Chinese explanations
type: user
---

User prefers Chinese explanations with concrete examples.
```

### 5.5 _rebuild_index：重建 MEMORY.md

代码位置：`s09_memory/code.py:84`

```python
def _rebuild_index():
    lines = []
    for f in sorted(MEMORY_DIR.glob("*.md")):
        if f.name == "MEMORY.md":
            continue
        ...
        lines.append(f"- [{name}]({f.name}) — {desc}")
    MEMORY_INDEX.write_text(...)
```

`MEMORY.md` 是记忆目录，不是全文：

```markdown
- [user-prefers-chinese](user-prefers-chinese.md) — User prefers Chinese explanations
```

### 5.6 build_system：把索引放进 SYSTEM

代码位置：`s09_memory/code.py:337`

```python
def build_system() -> str:
    index = read_memory_index()
    memories_section = f"\n\nMemories available:\n{index}" if index else ""
    return (
        f"You are a coding agent at {WORKDIR}."
        f"{memories_section}\n"
        "Relevant memories are injected below. Respect user preferences from memory.\n"
        "When the user says 'remember' or expresses a clear preference, extract it as a memory."
    )
```

这和 s07 很像：

```text
s07: skill catalog 放进 SYSTEM
s09: memory index 放进 SYSTEM
```

索引常驻，全文按需。

### 5.7 select_relevant_memories：选择相关记忆

代码位置：`s09_memory/code.py:132`

```python
def select_relevant_memories(messages: list, max_items: int = 5) -> list[str]:
```

它会：

```text
读取所有 memory 文件的 name/description
收集最近用户消息
让 LLM 从 memory catalog 中选相关项
失败时用关键词匹配降级
最多选 5 条
```

### 5.8 load_memories：读取相关记忆全文

代码位置：`s09_memory/code.py:207`

```python
def load_memories(messages: list) -> str:
    selected_files = select_relevant_memories(messages)
    ...
    parts = ["<relevant_memories>"]
    for filename in selected_files:
        content = read_memory_file(filename)
        if content:
            parts.append(content)
    parts.append("</relevant_memories>")
```

它返回：

```xml
<relevant_memories>
...相关记忆全文...
</relevant_memories>
```

### 5.9 把记忆注入当前请求

代码位置：`s09_memory/code.py:583`

```python
memories_content = load_memories(messages)
memory_turn = len(messages) - 1 if messages and isinstance(messages[-1].get("content"), str) else None
system = build_system()
```

代码位置：`s09_memory/code.py:606`

```python
request_messages = messages
if memories_content and memory_turn is not None and memory_turn < len(messages):
    request_messages = messages.copy()
    request_messages[memory_turn] = {
        **messages[memory_turn],
        "content": memories_content + "\n\n" + messages[memory_turn]["content"],
    }
```

注意它不直接改原 `messages`，而是构造 `request_messages` 发给 LLM。

LLM 实际看到：

```text
<relevant_memories>
...
</relevant_memories>

用户当前问题
```

### 5.10 extract_memories：每轮结束后提取新记忆

代码位置：`s09_memory/code.py:222`

```python
def extract_memories(messages: list):
```

它从最近对话中提取：

```text
用户偏好
用户约束
项目事实
用户反馈
```

并要求 LLM 返回 JSON：

```json
[
  {
    "name": "user-prefers-chinese",
    "type": "user",
    "description": "User prefers Chinese explanations",
    "body": "User prefers Chinese explanations with examples."
  }
]
```

之后调用 `write_memory_file()` 写入磁盘。

### 5.11 pre_compress：压缩前快照

代码位置：`s09_memory/code.py:592`

```python
pre_compress = [...]
```

为什么要保存压缩前版本？

因为 s08 可能把细节压没。  
提取记忆时，需要看到更完整的原始对话。

注意：

```text
pre_compress 临时占 Python RAM
不是长期保存
不是主 LLM 上下文
函数返回后会释放
```

### 5.12 consolidate_memories：整理记忆

代码位置：`s09_memory/code.py:287`

```python
def consolidate_memories():
    files = list_memory_files()
    if len(files) < CONSOLIDATE_THRESHOLD:
        return
```

记忆文件达到 10 个后，让 LLM：

```text
合并重复
删除过时
处理矛盾
保留重要偏好
```

### 5.13 s09 复习口诀

```text
messages 是短期记忆
.memory 是长期记忆
MEMORY.md 是索引
相关记忆按需注入
每轮结束提取新记忆
多了就 consolidate
```

---

## 6. s10：System Prompt，运行时组装

文件：`s10_system_prompt/code.py`

### 6.1 s10 解决的问题

前面章节会不断往 system prompt 里加内容：

```text
身份
工具
工作目录
技能目录
记忆索引
权限说明
上下文规则
```

如果一直用一个硬编码 `SYSTEM` 字符串，会越来越难维护。

s10 的方案：

```text
把 system prompt 拆成 section
根据当前真实 context 按需拼接
如果 context 没变，就用缓存
```

### 6.2 PROMPT_SECTIONS：定义可用片段

代码位置：`s10_system_prompt/code.py:42`

```python
PROMPT_SECTIONS = {
    "identity": "You are a coding agent. Act, don't explain.",
    "tools": "Available tools: bash, read_file, write_file.",
    "workspace": f"Working directory: {WORKDIR}",
    "memory": "Relevant memories are injected below when available.",
}
```

这就是把一整块 system prompt 拆成多个主题。

### 6.3 assemble_system_prompt：按 context 拼接

代码位置：`s10_system_prompt/code.py:50`

```python
def assemble_system_prompt(context: dict) -> str:
    sections = []
    sections.append(PROMPT_SECTIONS["identity"])
    sections.append(PROMPT_SECTIONS["tools"])
    sections.append(PROMPT_SECTIONS["workspace"])

    memories = context.get("memories", "")
    if memories:
        sections.append(f"Relevant memories:\n{memories}")

    return "\n\n".join(sections)
```

始终加载：

```text
identity
tools
workspace
```

条件加载：

```text
memory
```

如果 `.memory/MEMORY.md` 不存在或为空，`memory` section 不进入最终 prompt。

### 6.4 update_context：读取真实状态

代码位置：`s10_system_prompt/code.py:156`

```python
def update_context(context: dict, messages: list) -> dict:
    memories = ""
    if MEMORY_INDEX.exists():
        content = MEMORY_INDEX.read_text().strip()
        if content:
            memories = content
    return {
        "enabled_tools": list(TOOL_HANDLERS.keys()),
        "workspace": str(WORKDIR),
        "memories": memories,
    }
```

这里的 `context` 来自真实状态：

```text
实际注册了哪些工具
当前工作目录是什么
MEMORY.md 是否真的存在且有内容
```

不是根据用户消息里有没有关键词猜测。

### 6.5 get_system_prompt：缓存

代码位置：`s10_system_prompt/code.py:71`

```python
def get_system_prompt(context: dict) -> str:
    key = json.dumps(context, sort_keys=True, ensure_ascii=False, default=str)
    if key == _last_context_key and _last_prompt:
        return _last_prompt
    _last_context_key = key
    _last_prompt = assemble_system_prompt(context)
    return _last_prompt
```

作用：

```text
context 没变 -> 直接返回上次 prompt
context 变了 -> 重新组装 prompt
```

为什么用 `json.dumps`？

```text
dict/list 不能直接 hash
Python hash 不稳定
json.dumps(sort_keys=True) 能生成稳定字符串 key
```

注意：这是本地字符串缓存，不是 API prompt cache。

### 6.6 agent_loop 使用动态 system

代码位置：`s10_system_prompt/code.py:172`

```python
def agent_loop(messages: list, context: dict):
    system = get_system_prompt(context)
    while True:
        response = client.messages.create(
            model=MODEL,
            system=system,
            messages=messages,
            tools=TOOLS,
            max_tokens=8000
        )
```

以前是：

```python
system=SYSTEM
```

现在是：

```python
system=get_system_prompt(context)
```

工具执行后重新计算：

代码位置：`s10_system_prompt/code.py:195`

```python
context = update_context(context, messages)
system = get_system_prompt(context)
```

### 6.7 s10 复习口诀

```text
PROMPT_SECTIONS 定义材料
update_context 读取真实状态
assemble_system_prompt 按需拼装
get_system_prompt 做缓存
client.messages.create 发给 LLM
```

---

## 7. s06-s10 横向对比

| 章节 | 解决的问题 | 核心数据结构 | 关键函数 | 最终作用 |
| --- | --- | --- | --- | --- |
| s06 | 子任务过程污染主上下文 | 子 Agent 的局部 `messages[]` | `spawn_subagent()` | 主 Agent 只收总结 |
| s07 | 专业知识太多不能全塞 prompt | `SKILL_REGISTRY` | `list_skills()`, `load_skill()` | 技能目录常驻，全文按需 |
| s08 | `messages` 越来越长 | `messages` + transcript + tool output files | `snip_compact()`, `micro_compact()`, `compact_history()` | 长会话不爆上下文 |
| s09 | compact 会丢重要信息 | `.memory/*.md`, `MEMORY.md` | `extract_memories()`, `load_memories()` | 长期记住偏好和事实 |
| s10 | system prompt 越来越乱 | `PROMPT_SECTIONS`, `context` | `assemble_system_prompt()`, `get_system_prompt()` | prompt 动态组装 |

---

## 8. 三组容易混的概念

### 8.1 skill vs memory

```text
skill:
- 人提前写好的专业知识
- 存在 skills/
- 例子：code-review, pdf, mcp-builder
- 用 load_skill 按需加载

memory:
- Agent 从对话中积累的长期信息
- 存在 .memory/
- 例子：用户偏好、项目事实、反馈
- 每轮结束 extract_memories
```

### 8.2 compact vs memory

```text
compact:
- 解决当前 messages 太长
- 目标是减少上下文体积
- 可以丢细节

memory:
- 解决重要信息长期保存
- 目标是跨压缩、跨会话记住
- 不应该随便丢
```

一句话：

```text
compact 负责忘掉不重要的。
memory 负责记住重要的。
```

### 8.3 SYSTEM vs messages vs tool_result

```text
SYSTEM:
- 给模型的高优先级长期指令
- s07 放 skill catalog
- s09 放 memory index
- s10 动态组装

messages:
- 当前对话历史
- 会增长，会被 compact

tool_result:
- 工具执行结果
- 作为 user message 内容返回给模型
```

---

## 9. 五节合起来的执行想象

假设用户说：

```text
请 review 这个项目，并记住以后解释要用中文。
```

可能发生：

```text
1. s10 update_context()
   根据工具、workspace、memory 组装 system prompt

2. s07 system prompt 里有 skill catalog
   模型看到 code-review 技能

3. 模型调用 load_skill("code-review")
   完整 review 指南进入 tool_result

4. 任务很大时，模型调用 task
   s06 spawn_subagent 用独立 messages 做子任务

5. 读文件和工具输出很多
   s08 在下一轮 LLM 前压缩 messages

6. 用户说“以后用中文”
   s09 在 turn 结束后 extract_memories
   写入 .memory/user-prefers-chinese.md

7. 下次对话开始
   s10/s09 把 MEMORY.md 索引和相关记忆带回上下文
```

这就是 `s06-s10` 组合后的完整能力。

---

## 10. 口述版总结

如果别人问你：`s06-s10` 在讲什么？

可以这样回答：

> 这五节是在给 Agent 加上下文和知识管理能力。s06 用 subagent 把复杂子任务隔离出去，只把最终总结返回主 Agent。s07 用两级 skill loading，让模型先看到技能目录，需要时再加载完整技能文档。s08 在每次 LLM 调用前压缩 messages，避免上下文过长。s09 把用户偏好和项目事实写入 `.memory/`，形成跨压缩、跨会话的长期记忆。s10 把 system prompt 拆成 section，根据当前 context 运行时组装，避免硬编码一大段不可维护的 prompt。

更短版：

> s06-s10 的主题是：保护上下文、按需加载知识、长期保存重要信息、动态组装系统提示。

---

## 11. 自测题

### 题 1

主 Agent 调用 `task` 后，子 Agent 内部读了 10 个文件、跑了 5 次 bash。主 Agent 的 `messages` 会保存这 15 次详细过程吗？

答案：不会。主 Agent 只收到 `spawn_subagent()` 返回的最终总结。但子 Agent 的真实副作用，比如写文件，会保留。

### 题 2

`SKILL_REGISTRY` 模型能直接看到吗？

答案：不能。它是 Python 内存里的字典。必须通过 `list_skills()` 转成普通文本，再通过 `system=SYSTEM` 发给模型。

### 题 3

为什么 s07 不把所有 `SKILL.md` 全塞进 `SYSTEM`？

答案：因为大多数任务用不到所有技能，全文会浪费 token 并制造噪音。s07 只放目录，全文按需 `load_skill`。

### 题 4

s08 为什么要把大 tool_result 写到磁盘？

答案：大输出会撑爆上下文。写磁盘可以保留完整内容，同时在 `messages` 里只放路径和预览。

### 题 5

`compact_history()` 和 `extract_memories()` 的目标一样吗？

答案：不一样。`compact_history()` 是为了缩短当前上下文；`extract_memories()` 是为了长期保存重要偏好、约束、项目事实。

### 题 6

为什么 s09 要用 `pre_compress` 提取记忆？

答案：因为压缩后的 `messages` 可能已经丢了细节。`pre_compress` 是压缩前快照，能让记忆提取看到更完整的信息。

### 题 7

`.memory/MEMORY.md` 不存在时，s10 会把 memory section 加入 system prompt 吗？

答案：不会。`update_context()` 得到 `memories = ""`，`assemble_system_prompt()` 中 `if memories` 判断失败。

### 题 8

s10 的缓存和 API prompt cache 是一回事吗？

答案：不是。教学版 `get_system_prompt()` 只是避免本地重复拼接字符串；API prompt cache 是服务端缓存 prompt token 的机制。

---

## 12. 代码定位索引

### s06

- `s06_subagent/code.py:58` - `SUB_SYSTEM`
- `s06_subagent/code.py:182` - `SUB_TOOLS`
- `s06_subagent/code.py:196` - `SUB_HANDLERS`
- `s06_subagent/code.py:201` - `extract_text()`
- `s06_subagent/code.py:207` - `spawn_subagent()`
- `s06_subagent/code.py:210` - 子 Agent fresh `messages`
- `s06_subagent/code.py:212` - 子 Agent 30 轮上限
- `s06_subagent/code.py:249` - 只返回总结
- `s06_subagent/code.py:252` - `task` tool schema
- `s06_subagent/code.py:257` - `task` handler 映射

### s07

- `s07_skill_loading/code.py:47` - `SKILLS_DIR`
- `s07_skill_loading/code.py:53` - `_parse_frontmatter()`
- `s07_skill_loading/code.py:67` - `SKILL_REGISTRY`
- `s07_skill_loading/code.py:69` - `_scan_skills()`
- `s07_skill_loading/code.py:86` - `list_skills()`
- `s07_skill_loading/code.py:93` - `build_system()`
- `s07_skill_loading/code.py:102` - `SYSTEM = build_system()`
- `s07_skill_loading/code.py:269` - `load_skill()`
- `s07_skill_loading/code.py:297` - `load_skill` schema
- `s07_skill_loading/code.py:304` - `load_skill` handler 映射

### s08

- `s08_context_compact/code.py:265` - context constants
- `s08_context_compact/code.py:295` - `snip_compact()`
- `s08_context_compact/code.py:313` - `collect_tool_results()`
- `s08_context_compact/code.py:322` - `micro_compact()`
- `s08_context_compact/code.py:332` - `persist_large_output()`
- `s08_context_compact/code.py:339` - `tool_result_budget()`
- `s08_context_compact/code.py:357` - `write_transcript()`
- `s08_context_compact/code.py:364` - `summarize_history()`
- `s08_context_compact/code.py:375` - `compact_history()`
- `s08_context_compact/code.py:383` - `reactive_compact()`
- `s08_context_compact/code.py:416` - `compact` tool schema
- `s08_context_compact/code.py:454` - `agent_loop()` 压缩接入点
- `s08_context_compact/code.py:488` - 模型主动 compact

### s09

- `s09_memory/code.py:42` - `.memory/` 路径
- `s09_memory/code.py:56` - memory types
- `s09_memory/code.py:72` - `write_memory_file()`
- `s09_memory/code.py:84` - `_rebuild_index()`
- `s09_memory/code.py:98` - `read_memory_index()`
- `s09_memory/code.py:114` - `list_memory_files()`
- `s09_memory/code.py:132` - `select_relevant_memories()`
- `s09_memory/code.py:207` - `load_memories()`
- `s09_memory/code.py:222` - `extract_memories()`
- `s09_memory/code.py:287` - `consolidate_memories()`
- `s09_memory/code.py:337` - memory-aware `build_system()`
- `s09_memory/code.py:583` - memory loading in `agent_loop()`
- `s09_memory/code.py:592` - `pre_compress`
- `s09_memory/code.py:606` - memory injection into request
- `s09_memory/code.py:627` - extract memory after final response

### s10

- `s10_system_prompt/code.py:42` - `PROMPT_SECTIONS`
- `s10_system_prompt/code.py:50` - `assemble_system_prompt()`
- `s10_system_prompt/code.py:67` - prompt cache variables
- `s10_system_prompt/code.py:71` - `get_system_prompt()`
- `s10_system_prompt/code.py:156` - `update_context()`
- `s10_system_prompt/code.py:172` - dynamic prompt in `agent_loop()`
- `s10_system_prompt/code.py:195` - tool round 后重新计算 context/prompt
- `s10_system_prompt/code.py:204` - initial `context = update_context({}, [])`

---

## 13. 推荐复习顺序

第一次复习：

```text
1. 读第 1 节总主线
2. 读第 7 节横向对比
3. 读 s06、s07
4. 读 s08、s09
5. 读 s10
6. 做第 11 节自测题
```

第二次复习：

```text
1. 只看第 12 节代码索引
2. 跳到源码对应行
3. 口述每个函数在整个 Agent harness 中解决什么问题
```

第三次复习：

```text
自己画出这条线：

复杂任务 -> task -> subagent -> summary
专业知识 -> skill catalog -> load_skill
上下文太长 -> compact pipeline
重要信息 -> extract_memories -> .memory
系统提示 -> context -> prompt sections -> SYSTEM
```

能画出来，`s06-s10` 的骨架就掌握了。
