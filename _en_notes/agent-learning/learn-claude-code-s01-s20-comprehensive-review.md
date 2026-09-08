---
title: "learn-claude-code s01-s20: Comprehensive Agent Review Notes"
date: 2026-06-28 13:01:37 +0800
tags: [Agent, Claude Code, Coding Agent, Comprehensive Agent, Tool Use, Memory, Worktree, MCP, Agent Teams]
main_category: "Paper Reading"
sub_category: "Agent · learn-claude-code"
discipline: "Agent"
course: "learn-claude-code"
material_type: "Comprehensive Review"
description: "A comprehensive review of learn-claude-code s01-s20: from a minimal Agent Loop to tools, permissions, hooks, memory, teams, worktrees, MCP, and a complete coding agent harness."
lang: en
ref: "learn-claude-code-s01-s20-comprehensive-review"
---

Created: 2026-06-28  
Scope: `s01_agent_loop` through `s20_comprehensive`  
Goal: review how this project evolves step by step from a minimal Agent Loop into a complete coding agent harness.

---

## 0. In One Sentence

Reference: [learn-claude-code s01](https://learn.shareai.run/zh/s01/)

The model is responsible for:

> understanding, reasoning, deciding whether to call a tool, and deciding which tool to call.

Python is responsible for:

> maintaining `messages`, sending API requests, executing tools, checking permissions, triggering hooks, compacting context, saving state, managing the team, connecting external tools, and returning results to the model.

So the whole system is not "the model operating the computer by itself," but:

```text
The model outputs tool_use
Python executes the real tool
Python puts the tool_result back into messages
The model keeps reasoning based on the result
```

---

## 1. Overview: The s01-s20 Through-Line

These 20 lessons build up an increasingly complete Agent system step by step:

```text
s01  Minimal Agent Loop
s02  Multi-tool dispatch
s03  Permission checks
s04  Hooks extension points
s05  TodoWrite plan state
s06  Subagent context isolation
s07  Skill Loading, loading capabilities on demand
s08  Context Compact, compressing messages
s09  Memory, long-term memory
s10  System Prompt, dynamic assembly
s11  Error Recovery
s12  Task System, a persistent task graph
s13  Background Tasks
s14  Cron Scheduler, scheduled tasks
s15  Agent Teams, multi-agent teams
s16  Team Protocols, request-response protocol
s17  Autonomous Agents, autonomously claiming tasks
s18  Worktree Isolation, working directory isolation
s19  MCP Plugin, external tool integration
s20  Comprehensive Agent, all mechanisms folded into one loop
```

The ultimate through-line is still the s01 loop:

```text
User input
  -> messages
  -> client.messages.create(system, messages, tools)
  -> the model returns text or tool_use
  -> Python executes the tool
  -> tool_result is appended back to messages
  -> next round
```

The complexity of s20 is not another "brain," but a complete harness built around this loop.

---

## 2. One-Page Table Overview

| Module | What It Is | What It Solves | Why It Works This Way | Benefits | Costs |
|---|---|---|---|---|---|
| s01 | Basic Agent Loop | Lets the model request tools and Python execute them | Separates model reasoning from real execution | A minimal runnable closed loop | Few tools, weak security |
| s02 | Multi-tool system | How to dispatch as tools multiply | `TOOLS` for the model to see, `TOOL_HANDLERS` for Python to call | Easy to add tools | handler/schema must stay in sync |
| s03 | Permission system | Blocks dangerous tool calls | The security boundary must live at the Python layer | Does not rely solely on the prompt | Permission rules need maintenance |
| s04 | Hooks | Inserts extra logic at key points in the loop | Moves logging, permissions, auditing, etc. out of the main loop | Cleaner main loop, extensible | Feels abstract for beginners |
| s05 | TodoWrite | Manages the current task plan | Lets the model maintain task state explicitly | Complex tasks are less likely to drift | Only suits the current session |
| s06 | Subagent | Isolates the process of complex subtasks | The sub-agent uses its own `messages`; the main Agent only gets a summary | Saves main context | Sub-process details are not retained by default |
| s07 | Skill Loading | Loads skills on demand | Give a lightweight catalog first, load the full text when needed | Saves context | Requires skill metadata and a loading mechanism |
| s08 | Context Compact | Compresses long `messages` | The context window is finite | Prevents an overlong prompt | Details may be lost |
| s09 | Memory | Long-term memory | Both compaction and new sessions lose information | Important information can be reused long term | Requires extraction, filtering, and maintenance |
| s10 | System Prompt | Dynamically assembles the system prompt | The system prompt gets too long and messy | Clear sections, assembled per context | Assembly logic is more complex |
| s11 | Error Recovery | Recovery from API/context/output truncation | Different errors need different handling | A more stable Agent | Recovery strategies need classification |
| s12 | Task System | A persistent task graph | Todos only cover the current session, not project scale | Tasks can have dependencies, be claimed, and be saved | State management is more complex |
| s13 | Background Tasks | Runs slow tasks in the background | Long commands block the main loop | The main loop can keep working | Background results need async notification |
| s14 | Cron Scheduler | Scheduled tasks | Triggers tasks at a future time | Supports reminders and recurring tasks | Requires a queue and idle handling |
| s15 | Agent Teams | Multi-agent collaboration | One Agent doing everything is inefficient | Teammates work in parallel with independent context | Communication and state are more complex |
| s16 | Team Protocols | Request-response protocol | Teammate communication needs matching context | `request_id` avoids confusion | Requires maintaining pending state |
| s17 | Autonomous Agents | Teammates find work themselves | The Lead need not assign every task manually | Supports longer-running autonomy | Must prevent duplicate claims |
| s18 | Worktree Isolation | Working directory isolation | Multiple Agents editing code at once will conflict | Each task can run in its own worktree | Git/worktree management overhead |
| s19 | MCP Plugin | External tool integration | Tools cannot all be hard-coded locally | Tools are discovered dynamically via a protocol | Requires naming, permission, and connection management |
| s20 | Complete Agent | All mechanisms merged | A real Agent needs all these capabilities at once | Forms a complete harness | Highest code complexity |

---

## 3. Architectural Layers

The whole system can be split into 6 layers:

```text
Layer 1: Model interaction layer
client.messages.create, messages, system, tools, tool_use, tool_result

Layer 2: Tool execution layer
TOOLS schema, TOOL_HANDLERS, bash/read/write/edit/glob/MCP tools

Layer 3: Security and extension layer
permission, hook, PreToolUse, PostToolUse, Stop

Layer 4: Context and knowledge layer
compact, memory, skills, system prompt sections

Layer 5: Task and collaboration layer
todo_write, task graph, subagent, teammate, protocol, autonomous claim

Layer 6: Runtime and external integration layer
background tasks, cron, worktree, MCP
```

When interviewing or reviewing, do not treat each lesson as an isolated feature. They are all essentially answering the same question:

> For a model to operate a real environment over the long term, reliably and safely, which engineering capabilities does the Python harness need to supply?

---

## 4. Review Card for Each Lesson

### s01 Agent Loop

**What it is:** The minimal Agent loop.  
**What it solves:** The model cannot execute commands directly; it must execute through a Python proxy.  
**Why it works this way:** The LLM only outputs structured intent; real side effects are controlled by the program.  
**Core flow:**

```text
user message -> LLM -> tool_use -> Python handler -> tool_result -> LLM
```

**Benefits:** Simple structure, the foundation of every later mechanism.  
**Drawbacks:** Few tools, no permissions, compaction, or long-term state.

### s02 Tool Use

**What it is:** A multi-tool dispatch system.  
**What it solves:** When there is more than just `bash`, tools need unified registration and invocation.  
**Why it works this way:** The model reads the `TOOLS` schema; Python dispatches via `TOOL_HANDLERS`.  
**Core idea:**

```text
TOOLS = the tool descriptions shown to the model
TOOL_HANDLERS = the dictionary of functions Python actually calls
```

**Benefits:** Adding a tool only requires adding a schema and a handler.  
**Drawbacks:** Schema and handler parameters must correspond, or you get runtime errors.

### s03 Permission

**What it is:** A permission checking system.  
**What it solves:** The model may request dangerous commands, such as deleting files or system operations.  
**Why it works this way:** Security cannot live only in the prompt; it must intercept before Python executes.  
**Core idea:**

```text
The model may propose a dangerous tool call
Python may refuse to execute it
```

**Benefits:** Establishes a real security boundary.  
**Drawbacks:** A deny list and rules can never be inherently perfect; they need continuous maintenance.

### s04 Hooks

**What it is:** Lifecycle extension points.  
**What it solves:** If logging, permissions, auditing, and automatic handling all go into the main loop, the main loop gets messier and messier.  
**Why it works this way:** Register "extra functionality to run at a certain moment" into a hook.  
**Core idea:**

```text
hook = a dictionary + a list of functions + a for loop invoked at fixed moments
```

Typical moments:

```text
UserPromptSubmit: after user input
PreToolUse: before tool execution
PostToolUse: after tool execution
Stop: at the end of this round
```

**Benefits:** The main loop stays stable and new features can be bolted on.  
**Drawbacks:** The call chain is no longer fully explicit and is hard to trace at first.

### s05 TodoWrite

**What it is:** A plan and status tool within the current session.  
**What it solves:** In complex tasks the model easily forgets the plan or skips steps.  
**Why it works this way:** Have the model maintain a todo list explicitly.  
**Core idea:**

```text
todo_write is not a file tool
It is a state tool that lets the model manage the current task plan
```

**Benefits:** Complex tasks become better organized.  
**Drawbacks:** State is usually in memory, which is not the same as a cross-session task system.

### s06 Subagent

**What it is:** A one-shot sub-agent.  
**What it solves:** Complex subtasks pollute the main Agent's `messages`.  
**Why it works this way:** The sub-agent completes the task with its own `messages`; the main Agent only takes the final summary.  
**Core idea:**

```text
The main Agent's messages only stores the summary returned by the sub-agent
It does not store every detail of the sub-agent reading 10 files and running 5 commands
```

**Benefits:** Context isolation, a clearer main thread.  
**Drawbacks:** Sub-process details are lost by default unless deliberately recorded.

### s07 Skill Loading

**What it is:** On-demand skill loading.  
**What it solves:** Stuffing the full text of every skill into the system prompt makes it too long.  
**Why it works this way:** Give the model a lightweight catalog first, then `load_skill(name)` when needed.  
**Core chain:**

```text
SKILL.md
  -> _parse_frontmatter()
  -> SKILL_REGISTRY
  -> list_skills()
  -> build_system()
  -> client.messages.create(system=SYSTEM)
```

**Benefits:** Saves context, capabilities are extensible.  
**Drawbacks:** Requires maintaining skill metadata and a loading mechanism.

### s08 Context Compact

**What it is:** Context compaction.  
**What it solves:** `messages` keeps piling up and eventually exceeds the model's context window.  
**Why it works this way:** Compress old content before sending to the model while preserving the current working state.  
**Common approaches:**

```text
Persist large tool_results to files
Micro-compact old tool_results
Trim intermediate messages
Have the model summarize history when necessary
```

**Benefits:** The Agent can run longer.  
**Drawbacks:** Compaction always risks losing detail.

### s09 Memory

**What it is:** A long-term memory system.  
**What it solves:** Compaction loses information, and so does switching sessions.  
**Why it works this way:** Save long-term valuable information into `.memory/`.  
**Core idea:**

```text
compact mainly solves the current messages being too long
memory mainly solves persisting information long term across sessions
```

**Benefits:** Important preferences, project facts, and long-term constraints can be reused.  
**Drawbacks:** You must judge what is worth remembering, and wrong memories cause pollution.

### s10 System Prompt

**What it is:** Dynamic system prompt assembly.  
**What it solves:** A fixed, large prompt is hard to maintain, and not all of it is needed every round.  
**Why it works this way:** Split the prompt into sections, then assemble based on context.  
**Core idea:**

```text
assemble_system_prompt(context)
= identity + tools + workspace + memory + skills + MCP state ...
```

**Benefits:** Clear, maintainable, injectable on demand.  
**Drawbacks:** The prompt generation logic itself needs testing and maintenance.

### s11 Error Recovery

**What it is:** An error recovery mechanism.  
**What it solves:** API calls can fail, output can be truncated, and the prompt can be too long.  
**Why it works this way:** Different errors have different causes; you cannot retry them all the same way.  
**Core classification:**

```text
429 / 529: wait, then retry
max_tokens: raise max_tokens first, then retry
prompt_too_long: compact messages, then retry
```

**Benefits:** A more stable Agent.  
**Drawbacks:** Recovery strategies are complex and error classification must be accurate.

### s12 Task System

**What it is:** A persistent task system.  
**What it solves:** `todo_write` only suits the current session and is not project scale.  
**Why it works this way:** Use `.tasks/*.json` to store tasks, status, dependencies, and owner.  
**Core fields:**

```text
status: pending / in_progress / completed
owner: who claimed it
blockedBy: which tasks it depends on completing
```

**Benefits:** Supports long-running tasks, dependencies, and claiming.  
**Drawbacks:** Requires handling state consistency, for example preventing duplicate claims.

### s13 Background Tasks

**What it is:** A background task system.  
**What it solves:** Slow operations stall the main Agent loop.  
**Why it works this way:** Slow tools return a placeholder first, and the real result notifies later once the background work finishes.  
**Core flow:**

```text
slow bash -> executed on a background thread
the main loop immediately gets tool_result: started
background finishes -> <task_notification> injected into messages
```

**Benefits:** The main loop is not blocked by long tasks.  
**Drawbacks:** Results become asynchronous and the model has to handle delayed notifications.

### s14 Cron Scheduler

**What it is:** A scheduled task system.  
**What it solves:** The Agent needs to handle tasks that occur at some future point in time.  
**Why it works this way:** The scheduler only watches the clock; when the time comes the task enters a queue and the Agent handles it when idle.  
**Core structure:**

```text
CronJob: the task definition
Scheduler: checks the time
cron_queue: holds tasks that have fired but not been handled
agent_loop: turns tasks into messages
```

**Benefits:** Supports reminders and recurring tasks without directly interrupting the main loop.  
**Drawbacks:** Nothing runs automatically while the Python program is not running; only the durable definitions are preserved.

### s15 Agent Teams

**What it is:** A multi-agent team.  
**What it solves:** One Agent doing everything is slow and mixes up context.  
**Why it works this way:** The Lead Agent can spawn teammates, each with its own thread and `messages`.  
**Core idea:**

```text
The Lead and teammates do not share messages
They communicate via MessageBus / mailbox
```

**Benefits:** Parallelism, isolation, role specialization.  
**Drawbacks:** Communication, result aggregation, and lifecycle management get more complex.

### s16 Team Protocols

**What it is:** A team communication protocol.  
**What it solves:** Plain messages cannot express requests, approvals, and response matching.  
**Why it works this way:** Manage requests and responses with `request_id` and `pending_requests`.  
**Core idea:**

```text
request_id: matches a response back to the original request
pending_requests: in-memory state of requests awaiting a response
.mailboxes/*.jsonl: the message transport record
```

**Benefits:** Avoids mixing multiple requests together.  
**Drawbacks:** Requires maintaining protocol types and a state machine.

### s17 Autonomous Agents

**What it is:** Autonomous teammates.  
**What it solves:** The Lead does not want to hand out every task manually.  
**Why it works this way:** When idle, a teammate checks its inbox first, then scans for claimable tasks.  
**Core flow:**

```text
idle_poll()
  -> read_inbox()
  -> scan_unclaimed_tasks()
  -> claim_task()
  -> inject <auto-claimed> into the teammate's messages
```

**Benefits:** Teammates can find work on their own; the system is more automated.  
**Drawbacks:** Must strictly check `owner`, `status`, and `blockedBy` to prevent two agents grabbing the same task.

### s18 Worktree Isolation

**What it is:** Git worktree working directory isolation.  
**What it solves:** Multiple Agents editing the same directory at once can conflict.  
**Why it works this way:** Bind each task to its own worktree; once a teammate claims it, the tools' cwd switches to that directory.  
**Core idea:**

```text
bind_task_to_worktree only binds the directory
claim_task is what marks the task as claimed by a given agent
Python must genuinely switch the cwd of bash/read/write over to it
```

**Benefits:** Parallel development is safer and changes are isolated.  
**Drawbacks:** Requires managing branches, cleaning up worktrees, and avoiding accidental deletion of uncommitted changes.

### s19 MCP Plugin

**What it is:** An external tool integration protocol.  
**What it solves:** Tools cannot all be hard-coded into the Agent's local code.  
**Why it works this way:** An external MCP server publishes tools, and the Agent discovers them dynamically after connecting.  
**Core flow:**

```text
connect_mcp("docs")
  -> MCPClient
  -> mcp_clients
  -> assemble_tool_pool()
  -> mcp__docs__search
```

**Benefits:** External capabilities are pluggable and the tool pool expands dynamically.  
**Drawbacks:** Requires handling name collisions, permissions, connection state, and server reliability.

### s20 Comprehensive Agent

**What it is:** The complete Agent harness.  
**What it solves:** The first 19 lessons each cover one mechanism, but a real Agent needs all of them at once.  
**Why it works this way:** Hang every component back onto the same `agent_loop`.  
**Core idea:**

```text
Many mechanisms, one loop
```

One round of the s20 loop:

```text
inject cron
inject background notification
todo reminder
prepare_context compaction
update_context
assemble_tool_pool
call_llm
error recovery
PreToolUse hook / permission
execute the tool or a background task
PostToolUse hook
append tool_result
continue to the next round
```

**Benefits:** Shows how a complete system runs.  
**Drawbacks:** The largest amount of code; you have to break it down by position in the main loop to understand it.

---

## 5. Easily Confused Concepts

### 5.1 `tools schema` vs `handler`

```text
tools schema: the tool format shown to the model, telling it tool names, parameter types, and purposes.
handler: the function Python actually executes internally.
```

The model never gets the Python function directly.  
The model only sees the schema, then returns a `tool_use`.  
Python looks up the handler by `tool_use.name`.

### 5.2 `todo_write` vs `Task System`

```text
todo_write: a lightweight plan within the current session.
Task System: a cross-session task graph that supports dependencies, claiming, and persistence.
```

### 5.3 `compact` vs `memory`

```text
compact: solves the current messages being too long.
memory: persists long-term valuable information for reuse across sessions.
```

### 5.4 `subagent` vs `teammate`

```text
subagent: one-shot; the main Agent only takes the final summary.
teammate: a long-lived thread with its own inbox, messages, and role, able to keep working.
```

### 5.5 `mailbox` vs `pending_requests`

```text
mailbox: the message transport channel, storing JSONL messages.
pending_requests: the in-memory protocol state table recording which requests are still awaiting a response.
```

### 5.6 `bind_task_to_worktree` vs `claim_task`

```text
bind_task_to_worktree: which directory this task should be done in.
claim_task: which agent is responsible for this task, moving it into in_progress.
```

### 5.7 `connect_mcp` vs `assemble_tool_pool`

```text
connect_mcp: connects to an external tool source.
assemble_tool_pool: turns external tools into a tool pool the model can see and Python can call.
```

---

## 6. The s20 Main Loop Diagram

```text
User input
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

## 7. Overall Assessment of Benefits and Costs

### 7.1 Overall Benefits

- A clear division of labor between the model and the execution environment.
- The tool system is extensible.
- The permission boundary lives at the Python layer, which is more reliable.
- Hooks let the main loop be extended without frequent modification.
- compact/memory/skills make context more controllable.
- task/team/worktree support more complex engineering collaboration.
- background/cron let the Agent handle slow tasks and future tasks.
- MCP turns external tool integration into a protocol problem.

### 7.2 Overall Costs

- More and more state: `messages`, `context`, `tasks`, `memory`, `mailboxes`, `mcp_clients`, `background_tasks`.
- Harder debugging: a single model behavior may be influenced jointly by the system prompt, memory, hooks, permissions, and tools.
- More complex security: local tools, MCP tools, background tasks, and worktrees all need permission boundaries.
- Harder consistency: task state, teammate state, protocol state, and cron state can all fall out of sync.
- Compaction and memory affect the model's cognition: compaction may lose detail, and memory may introduce stale information.

---

## 8. Project Summary

```text
This project implements a coding agent harness from scratch.
At the very bottom is the Agent Loop: based on messages and the tools schema, the model decides whether to issue a tool_use; Python executes the tool via the handler and puts the tool_result back into messages.

The later chapters do not change this core loop. They add engineering capabilities around it:
permissions and hooks handle security and extensibility;
todo, task, subagent, and team handle complex task management;
compact, memory, skill, and system prompt handle context and capability management;
background, cron, worktree, and MCP handle the real runtime environment, concurrent collaboration, and external tool integration.

In the end s20 folds these mechanisms back into a single main loop, so the core can be summarized as:
many mechanisms, one loop.
```