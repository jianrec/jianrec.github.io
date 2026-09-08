---
title: "learn-claude-code s01-s05: Agent Foundation Review Notes"
date: 2026-06-26 16:06:23 +0800
tags: [Agent, Claude Code, Coding Agent, Tool Use, Permission, Hooks, TodoWrite, Source Code Study]
main_category: "Paper Reading"
sub_category: "Agent · learn-claude-code"
discipline: "Agent"
course: "learn-claude-code"
material_type: "Source Code Study"
description: "Reviewing learn-claude-code s01-s05: from the minimal agent loop, multi-tool dispatch, the permission pipeline, and the hook system, all the way to task state management with todo_write."
lang: en
---

Created: 2026-06-26  
Scope: `s01_agent_loop` through `s05_todo_write`  
Goal: review the core structure, execution principles, code layering, and common points of confusion in a minimal coding agent harness.

---

## 0. Memorize These Five Sentences First

1. `s01`: the minimal closed loop of an Agent is `LLM -> tool_use -> Python executes the tool -> tool_result -> LLM`.
2. `s02`: when adding a new tool, you do not touch the main loop; you only add a `TOOLS` schema, a tool function, and a `TOOL_HANDLERS` mapping.
3. `s03`: permissions must be intercepted at the Python execution layer; you cannot rely on the prompt to make the model behave.
4. `s04`: a hook is a lifecycle insertion point reserved by the program, used to move permission, logging, statistics, and similar logic out of the main loop.
5. `s05`: `todo_write` is a planning and state tool that lets the model manage tasks explicitly instead of just "keeping them in its head."

---

## 1. The Biggest Mental Model: Model + Harness

This project is not about "writing an intelligent brain." The intelligence comes from the model itself. What the code does is provide a harness — an environment in which the model can act.

```text
The Model is responsible for:
- Understanding the user's task
- Judging what to do next
- Deciding whether to call a tool
- Continuing to reason based on tool_result

The Harness is responsible for:
- Telling the model which tools are available
- Receiving the tool_use returned by the model
- Executing the real Python functions
- Controlling permissions and safety boundaries
- Putting tool results back into messages
```

So always keep this in mind:

```text
The model is responsible for "deciding"
Python is responsible for "executing"
```

The model does not actually read files, write files, or run bash by itself. It only returns a structured request:

```text
Which tool do I want to call?
What are the arguments?
```

Only after Python receives it does a real local function actually run.

---

## 2. The Overall Agent Loop Execution Flow

The main loop is essentially unchanged from s01 through s05; only the machinery around it keeps growing.

```text
User input
  |
  v
messages.append({"role": "user", "content": query})
  |
  v
client.messages.create(model, system, messages, tools)
  |
  v
Model returns a response
  |
  +-- stop_reason != "tool_use"
  |      |
  |      v
  |    Final answer, agent_loop ends
  |
  +-- stop_reason == "tool_use"
         |
         v
      Iterate over response.content
         |
         v
      Find block.type == "tool_use"
         |
         v
      Python executes the corresponding tool function
         |
         v
      results.append({"type": "tool_result", ...})
         |
         v
      messages.append({"role": "user", "content": results})
         |
         v
      Back to the LLM, continue the loop
```

The most central loop code lives at `s01_agent_loop/code.py:85`:

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

The line most worth staring at repeatedly is the last one:

```python
messages.append({"role": "user", "content": results})
```

The tool result is wrapped as a new user message and sent back to the model. Only after seeing the tool result does the model know what to do next.

---

## 3. s01: The Minimal Agent Loop

File: `s01_agent_loop/code.py`

### 3.1 What Capability s01 Adds

s01 has only one tool: `bash`.

The model can ask Python to execute a shell command. For example, the model returns:

```python
block.name = "bash"
block.input = {"command": "ls"}
```

Python then executes:

```python
run_bash("ls")
```

### 3.2 The TOOLS Schema Is the Manual Written for the Model

Code location: `s01_agent_loop/code.py:57`

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

It tells the model three things:

```text
Tool name: bash
Purpose: run a shell command
Arguments: command is required, and command must be a string
```

Note: what the model sees is this schema, not the `run_bash()` function itself.

### 3.3 run_bash Is Where the Action Actually Happens

Code location: `s01_agent_loop/code.py:69`

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

This snippet has three layers:

```text
Layer one: dangerous command filtering
Layer two: subprocess.run actually executes the shell command
Layer three: try/except turns exceptions into strings returned to the model
```

Why return a string instead of raising an exception directly?

Because the Agent loop needs to keep running. A tool failure should also become an observation the model can understand:

```text
Error: Timeout (120s)
Error: Dangerous command blocked
```

Once the model sees the error, it can try a different approach and continue.

### 3.4 The Design Value of s01

s01 proves one thing:

```text
As long as you have:
1. messages
2. tools schema
3. client.messages.create()
4. tool_result sent back

you have a minimal agent loop.
```

This is the foundation for every later chapter.

---

## 4. s02: From a Single Tool to Multiple Tools

File: `s02_tool_use/code.py`

### 4.1 What Capability s02 Adds

s01 only had `bash`. s02 adds four more tools:

```text
read_file  - read a file
write_file - write a file
edit_file  - replace a piece of text in a file
glob       - find files by wildcard pattern
```

Once the number of tools grows, the main loop can no longer be hardcoded:

```python
run_bash(block.input["command"])
```

So s02 introduces `TOOL_HANDLERS`.

### 4.2 safe_path: The Safe Entry Point for All File Operations

Code location: `s02_tool_use/code.py:66`

```python
def safe_path(p: str) -> Path:
    path = (WORKDIR / p).resolve()
    if not path.is_relative_to(WORKDIR):
        raise ValueError(f"Path escapes workspace: {p}")
    return path
```

The problem it solves: the model might ask to read or write a path outside the workspace.

For example:

```text
../../../etc/passwd
/Users/someone/.ssh/id_rsa
```

`safe_path()` resolves the path into an absolute path and then checks whether it is still inside `WORKDIR`.

So all file tools call it first:

```python
safe_path(path).read_text()
safe_path(path).write_text(content)
```

Design principle:

```text
Never trust the path the model sends.
Do a boundary check before every real file operation.
```

### 4.3 The Responsibilities of the Four File Tools

Code location: `s02_tool_use/code.py:73`

```text
run_read(path, limit)
- Reads the file content
- limit can cap the number of lines read

run_write(path, content)
- Writes the full file content
- Creates the parent directory with mkdir if it does not exist

run_edit(path, old_text, new_text)
- Replaces only the first occurrence of old_text
- Returns an error if old_text does not exist

run_glob(pattern)
- Finds files by wildcard pattern
- Returns only a list of paths, does not read file contents
```

The `replace(old_text, new_text, 1)` inside `run_edit()` matters a lot:

```python
text.replace(old_text, new_text, 1)
```

The trailing `1` means only the first occurrence is replaced, which avoids accidentally changing several places at once.

### 4.4 TOOL_HANDLERS: The Routing Table from Tool Name to Function

Code location: `s02_tool_use/code.py:138`

```python
TOOL_HANDLERS = {
    "bash": run_bash,
    "read_file": run_read,
    "write_file": run_write,
    "edit_file": run_edit,
    "glob": run_glob,
}
```

Its purpose is:

```text
The model returns a tool name
Python finds the real function from that tool name
```

The main loop becomes:

Code location: `s02_tool_use/code.py:165`

```python
handler = TOOL_HANDLERS.get(block.name)
output = handler(**block.input) if handler else f"Unknown: {block.name}"
```

There are two key points here:

1. `block.name` is the tool name the model chose.
2. `**block.input` expands the dictionary into function arguments.

For example:

```python
block.name = "read_file"
block.input = {"path": "README.md", "limit": 20}
```

is actually equivalent to:

```python
run_read(path="README.md", limit=20)
```

### 4.5 The Design Value of s02

s02 establishes the standard method for extending tools:

```text
Adding a new tool takes three steps:

1. Write the real Python function
2. Write a schema in TOOLS for the model to see
3. Map the name to the function in TOOL_HANDLERS
```

The main loop does not need to understand the details of each tool; it only dispatches.

---

## 5. s03: The Permission System

File: `s03_permission/code.py`

### 5.1 The Problem s03 Solves

s02 gave the model the ability to read and write files and run commands, but the greater the capability, the greater the risk.

The model might request:

```text
rm file
chmod 777
writing to a path outside the workspace
sudo ...
```

So s03 adds a permission pipeline before a tool actually executes.

Core principle:

```text
Safety cannot rely on the system prompt alone.
Safety must be intercepted before Python executes.
```

### 5.2 Three Permission Gates

Code location: `s03_permission/code.py:148`

Gate one: the hard deny list.

```python
DENY_LIST = ["rm -rf /", "sudo", "shutdown", "reboot", "mkfs", "dd if=", "> /dev/sda"]
```

Once one of these commands matches, it is rejected outright without asking the user.

Gate two: rule checks.

Code location: `s03_permission/code.py:159`

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

These rules are not permanent bans; they require user confirmation.

Gate three: user approval.

Code location: `s03_permission/code.py:176`

```python
choice = input("   Allow? [y/N] ").strip().lower()
return "allow" if choice in ("y", "yes") else "deny"
```

### 5.3 check_permission Is the Permission Entry Point

Code location: `s03_permission/code.py:184`

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

What it receives is a `block`, that is, the tool_use returned by the model.

Why pass `block` instead of just the command?

Because different tools need different checks:

```text
bash      -> look at command
write_file -> look at path
edit_file  -> look at path, old_text, new_text
```

### 5.4 The Permission Insertion Point

Code location: `s03_permission/code.py:220`

```python
if not check_permission(block):
    results.append({
        "type": "tool_result",
        "tool_use_id": block.id,
        "content": "Permission denied."
    })
    continue
```

Meaning:

```text
Check permissions before executing a tool
If not allowed:
  do not run the handler
  return Permission denied to the model
  continue with the next tool call
```

Note: even when a tool is denied, you still have to return a `tool_result`. Because the model has already issued a tool_use, the protocol requires a corresponding result for it.

### 5.5 The Design Value of s03

s03 puts the Agent's ability to act inside a safety boundary.

This is a very important engineering principle:

```text
The model can propose an action request.
Python must decide whether that action can actually happen.
```

---

## 6. s04: The Hooks Extension System

File: `s04_hooks/code.py`

### 6.1 The Problem s04 Solves

s03 already has permission checks, but the permission logic is written directly into `agent_loop`.

If later you also want to add:

```text
Log every tool call
Count how many tools were used
Check whether the output is too large
Add context before the user input
Automatically clean up after tool execution
```

and you keep stuffing all of it into `agent_loop`, the main loop will get messier and messier.

The s04 approach is:

```text
The main loop keeps only a few "trigger points"
Concrete feature functions are registered onto those trigger points
```

This is what a hook is.

### 6.2 The Essence of a Hook

Code location: `s04_hooks/code.py:159`

```python
HOOKS = {"UserPromptSubmit": [], "PreToolUse": [], "PostToolUse": [], "Stop": []}
```

This is just an ordinary dictionary:

```text
key   = the event name, that is, the moment
value = a list of functions, that is, the callbacks to run at that moment
```

You can think of it as:

```text
The UserPromptSubmit drawer: functions to run when the user submits input
The PreToolUse drawer: functions to run before a tool executes
The PostToolUse drawer: functions to run after a tool executes
The Stop drawer: functions to run before the Agent stops
```

### 6.3 register_hook: Putting a Function into a Drawer

Code location: `s04_hooks/code.py:161`

```python
def register_hook(event: str, callback):
    HOOKS[event].append(callback)
```

Registration does not execute the function. It only stores the function object.

For example:

Code location: `s04_hooks/code.py:225`

```python
register_hook("PreToolUse", permission_hook)
register_hook("PreToolUse", log_hook)
```

means:

```text
From now on, every time PreToolUse fires,
run permission_hook and then log_hook in order.
```

### 6.4 trigger_hooks: Actual Execution Only When the Moment Arrives

Code location: `s04_hooks/code.py:164`

```python
def trigger_hooks(event: str, *args):
    for callback in HOOKS[event]:
        result = callback(*args)
        if result is not None:
            return result
    return None
```

This snippet is critical. What it actually does is:

```text
Take the list of functions in HOOKS[event]
Call them one by one
If some hook returns non-None, return immediately
If all hooks return None, nothing intercepted
```

If right now:

```python
HOOKS["PreToolUse"] = [permission_hook, log_hook]
```

then:

```python
trigger_hooks("PreToolUse", block)
```

is equivalent to:

```python
result = permission_hook(block)
if result is not None:
    return result

result = log_hook(block)
if result is not None:
    return result

return None
```

So hooks are not mysterious. They are simply:

```text
a dictionary + a list of functions + a for loop that calls them
```

### 6.5 The Four Hook Moments

The code has four kinds of events:

```text
UserPromptSubmit
- Fires after user input, before it is actually added to messages
- Currently used to print the working directory

PreToolUse
- Fires before a tool executes
- Currently used for permission checks and logging

PostToolUse
- Fires after a tool executes
- Currently used to check for large output

Stop
- Fires when the model stops calling tools and agent_loop is about to exit
- Currently used to print tool call statistics
```

Trigger locations:

```python
trigger_hooks("UserPromptSubmit", query)       # s04_hooks/code.py:287
blocked = trigger_hooks("PreToolUse", block)  # s04_hooks/code.py:259
trigger_hooks("PostToolUse", block, output)   # s04_hooks/code.py:268
force = trigger_hooks("Stop", messages)       # s04_hooks/code.py:247
```

### 6.6 The Difference Between hook and tool

This is the easiest place to get confused.

```text
tool:
- A capability shown to the model
- The model can actively request to call it
- Has a schema
- Executed through TOOL_HANDLERS
- Examples: bash, read_file, write_file, edit_file, glob

hook:
- A lifecycle extension point inside the Python program
- The model usually does not know about it
- Has no tool schema
- Called by trigger_hooks at fixed moments
- Examples: permission_hook, log_hook, summary_hook
```

In one sentence:

```text
A tool is the Agent's "hand"; it does the work.
A hook is a "checkpoint" in the flow; it inserts logic before and after the work.
```

### 6.7 The Design Value of s04

s04 keeps the main loop stable:

```python
blocked = trigger_hooks("PreToolUse", block)
...
trigger_hooks("PostToolUse", block, output)
```

To add a feature later, you do not need to keep modifying `agent_loop`; you only need:

```python
def new_hook(...):
    ...

register_hook("some_event", new_hook)
```

That is the value of extension points.

---

## 7. s05: The TodoWrite Planning Tool

File: `s05_todo_write/code.py`

### 7.1 The Problem s05 Solves

By now the Agent can call tools, check permissions, and run hooks.

But there is still one problem: for a complex task without an explicit plan, the model easily forgets things as it goes, or drifts off target after finishing part of the work.

s05 introduces `todo_write` so the model writes a task list before a multi-step task and updates its status during execution.

### 7.2 The SYSTEM Prompt Reminds the Model to Plan First

Code location: `s05_todo_write/code.py:53`

```python
SYSTEM = (
    f"You are a coding agent at {WORKDIR}. "
    "Before starting any multi-step task, use todo_write to plan your steps. "
    "Update status as you go."
)
```

It is not a hard rule but behavioral guidance:

```text
Before a multi-step task, plan with todo_write first.
During execution, update task status.
```

What actually lets the model call `todo_write` is the tool schema that follows.

### 7.3 CURRENT_TODOS: Task State in Memory

Code location: `s05_todo_write/code.py:50`

```python
CURRENT_TODOS: list[dict] = []
```

This means the current task list exists only in the program's memory.

Characteristics:

```text
Valid while the program is running
Gone once the program exits
Not written to a file
Not a database
```

### 7.4 _normalize_todos: Validating the Data the Model Sends

Code location: `s05_todo_write/code.py:124`

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

What it does is input sanitization and format validation.

Why is it needed?

Although the schema tells the model that `todos` should be an array, the model's output may still be imperfect. The execution layer had better check once more.

It accepts two forms:

```text
1. A real list
2. A JSON string or a Python literal string
```

Each todo must be:

```python
{
    "content": "...",
    "status": "pending" | "in_progress" | "completed"
}
```

The leading `_` in the function name is a convention: this is an internal helper function, not a main external interface.

### 7.5 run_todo_write: Saving and Displaying Tasks

Code location: `s05_todo_write/code.py:144`

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

It does three things here:

```text
1. Validate todos
2. Update the global variable CURRENT_TODOS
3. Print the current task list and return the update result
```

Note: `todo_write` does not execute tasks.

It only records the plan:

```text
pending      not started yet
in_progress in progress
completed   finished
```

### 7.6 The todo_write schema

Code location: `s05_todo_write/code.py:169`

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

The schema tells the model:

```text
todo_write needs a todos argument
todos is an array
Every item in the array has content and status
status can only be one of three values
```

### 7.7 The reminder Mechanism

Code location: `s05_todo_write/code.py:235`

```python
rounds_since_todo = 0
```

Code location: `s05_todo_write/code.py:241`

```python
if rounds_since_todo >= 3 and messages:
    messages.append({"role": "user",
                     "content": "<reminder>Update your todos.</reminder>"})
    rounds_since_todo = 0
```

If the model calls tools for several rounds in a row without updating its todos, the program inserts a reminder into messages:

```text
<reminder>Update your todos.</reminder>
```

When the model calls `todo_write`, the counter resets to zero:

Code location: `s05_todo_write/code.py:276`

```python
if block.name == "todo_write":
    rounds_since_todo = 0
```

### 7.8 The Design Value of s05

s05 makes the Agent not just "able to do things" but also start "managing task state."

```text
s01-s04: make the Agent able to act, safely and extensibly.
s05: make the Agent's actions planned.
```

---

## 8. How the Five Sections Evolve

| Section | What Is Added | Core Change | Major Main-Loop Rewrite? |
| --- | --- | --- | --- |
| s01 | `bash` | Minimal agent loop | Yes, it establishes the foundation |
| s02 | Multiple tools + `TOOL_HANDLERS` | From hardcoded tools to generic dispatch | Small change in the tool execution part |
| s03 | permission pipeline | A safety gate before execution | Inserts the permission check |
| s04 | hook system | Moves extension logic out of the main loop | Replaces hardcoded checks with triggers |
| s05 | `todo_write` | Adds explicit planning and state management | Only adds the reminder counter |

You can review along this line:

```text
can act -> acts in more ways -> checks before acting -> the checking logic is extensible -> acts with a plan
```

---

## 9. The Four Most Easily Confused Concepts

### 9.1 schema vs Python Function

```text
schema:
- The description shown to the model
- Spells out the tool name, description, and argument structure
- Executes nothing

Python function:
- The real action the program executes
- For example run_bash, run_read, run_todo_write
```

The model only sees the schema. Python finds the function via the tool name.

### 9.2 tool_use vs tool_result

```text
tool_use:
- A request issued by the model
- It means: I want to call a certain tool

tool_result:
- The result after Python executes the tool
- It means: the tool has run, here is the result
```

The flow:

```text
Model -> tool_use
Python -> tool_result
Model -> continues based on tool_result
```

### 9.3 tool vs hook

```text
tool:
- Actively called by the model
- For example bash/read_file/todo_write
- It is an action capability

hook:
- Called automatically by the program at fixed moments
- For example permission_hook/log_hook
- It is a flow extension capability
```

### 9.4 permission vs safe_path

```text
permission:
- A policy-layer check before a tool executes
- Can ask the user whether to allow it
- Appears starting from s03

safe_path:
- A hard boundary inside file operations
- Prevents paths from escaping the workspace
- Appears starting from s02
```

The two can coexist:

```text
permission first decides "whether the attempt is allowed"
safe_path then guarantees "the file path cannot go out of bounds"
```

---

## 10. A Method for Reading Agent Code

Whenever you see new Agent code from now on, you can read it by asking these 7 questions:

1. Where is the entry point? How does user input get into `messages`?
2. What arguments are passed to `client.messages.create()`?
3. Which tools does `TOOLS` expose to the model?
4. Does every tool have a corresponding Python handler?
5. Where is `tool_use` iterated over and executed?
6. Is there a permission/hook/sandbox before execution?
7. How does `tool_result` get back into `messages`?

These 7 questions will help you quickly understand most tool-use agents.

---

## 11. The Spoken Summary

If someone asks you: what are s01-s05 of this project about?

You can answer like this:

> These five sections build a minimal coding agent harness. s01 first implements the most central agent loop: the model returns a tool_use, Python executes the tool, and the tool_result is fed back to the model. s02 extends the single bash tool into a multi-tool system and uses TOOL_HANDLERS for dispatch. s03 adds a permission pipeline before tool execution so that dangerous operations do not run directly. s04 extracts permission, logging, statistics, and other extension logic into hooks so the main loop stays clean. s05 adds todo_write so the model explicitly plans and updates state during complex tasks.

A shorter version:

> The model makes decisions; the Python harness executes tools, controls permissions, and manages context and state.

---

## 12. Self-Test Questions

### Question 1

Can the model directly call the Python function `run_bash()`?

Answer: no. The model can only see the `TOOLS` schema. It returns a `tool_use`, and Python decides whether to call `run_bash()` based on `block.name` and `block.input`.

### Question 2

Why does s02 need to add `TOOL_HANDLERS`?

Answer: because once there are more tools, the main loop cannot hardcode `run_bash()`. `TOOL_HANDLERS` maps tool names to real functions so the main loop can dispatch uniformly.

### Question 3

If the model requests a dangerous command, why can't the system prompt alone stop it?

Answer: a prompt is behavioral guidance, not a security boundary. The real security boundary must be checked before Python executes, for example in `check_permission()` or `permission_hook()`.

### Question 4

Is a hook a tool?

Answer: no. A tool is an action the model can request; a hook is a helper function the Python program calls automatically at fixed points in the flow.

### Question 5

Does `todo_write` execute tasks automatically?

Answer: no. It only records and displays the task plan; actual execution still relies on tools like bash/read/write/edit.

### Question 6

Why does a denied tool call still need to return a `tool_result`?

Answer: because the model has already issued a tool_use, and the protocol requires a corresponding result. Even if the result is `"Permission denied."`, the model needs to know the outcome of that tool call.

### Question 7

What does `**block.input` mean?

Answer: it expands the dictionary into function arguments. For example, `{"path": "a.txt", "limit": 10}` becomes `run_read(path="a.txt", limit=10)`.

---

## 13. Code Location Index

### s01

- `s01_agent_loop/code.py:57` - `bash` tool schema
- `s01_agent_loop/code.py:69` - `run_bash()`
- `s01_agent_loop/code.py:85` - the minimal `agent_loop()`
- `s01_agent_loop/code.py:87` - `client.messages.create()`
- `s01_agent_loop/code.py:106` - constructing `tool_result`
- `s01_agent_loop/code.py:113` - the tool result goes back into `messages`

### s02

- `s02_tool_use/code.py:66` - `safe_path()`
- `s02_tool_use/code.py:73` - `run_read()`
- `s02_tool_use/code.py:83` - `run_write()`
- `s02_tool_use/code.py:93` - `run_edit()`
- `s02_tool_use/code.py:105` - `run_glob()`
- `s02_tool_use/code.py:121` - the multi-tool `TOOLS`
- `s02_tool_use/code.py:138` - `TOOL_HANDLERS`
- `s02_tool_use/code.py:165` - generic tool dispatch

### s03

- `s03_permission/code.py:148` - `DENY_LIST`
- `s03_permission/code.py:159` - `PERMISSION_RULES`
- `s03_permission/code.py:176` - `ask_user()`
- `s03_permission/code.py:184` - `check_permission()`
- `s03_permission/code.py:220` - the permission interception point before executing a tool

### s04

- `s04_hooks/code.py:159` - `HOOKS`
- `s04_hooks/code.py:161` - `register_hook()`
- `s04_hooks/code.py:164` - `trigger_hooks()`
- `s04_hooks/code.py:176` - `permission_hook()`
- `s04_hooks/code.py:200` - `log_hook()`
- `s04_hooks/code.py:206` - `large_output_hook()`
- `s04_hooks/code.py:218` - `summary_hook()`
- `s04_hooks/code.py:225` - hook registration
- `s04_hooks/code.py:259` - `PreToolUse` trigger
- `s04_hooks/code.py:268` - `PostToolUse` trigger
- `s04_hooks/code.py:287` - `UserPromptSubmit` trigger

### s05

- `s05_todo_write/code.py:50` - `CURRENT_TODOS`
- `s05_todo_write/code.py:53` - the `SYSTEM` prompt that reminds the model to plan first
- `s05_todo_write/code.py:124` - `_normalize_todos()`
- `s05_todo_write/code.py:144` - `run_todo_write()`
- `s05_todo_write/code.py:169` - `todo_write` schema
- `s05_todo_write/code.py:173` - adding `todo_write` to `TOOL_HANDLERS`
- `s05_todo_write/code.py:235` - `rounds_since_todo`
- `s05_todo_write/code.py:241` - reminder injection
- `s05_todo_write/code.py:276` - resetting the counter after `todo_write` is called

---

## 14. Recommended Review Order

First review pass:

```text
1. Read section 2, the overall Agent Loop execution flow
2. Read section 9, the four easily confused concepts
3. Read s01 and s02
4. Read s03 and s04
5. Read s05
6. Do the self-test questions in section 12
```

Second review pass:

```text
1. Look only at the code index in section 13
2. Jump to the corresponding source locations
3. Explain out loud what each piece of code does in the flow
```

Third review pass:

```text
Try to draw this line without looking at the notes:

User input
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

If you can draw it, you have mastered the skeleton of s01-s05.