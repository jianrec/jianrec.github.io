---
title: "learn-claude-code s11-s15: From Fault Tolerance to Multi-Agent Teams"
date: 2026-06-27 21:44:51 +0800
tags: [Agent, Claude Code, Coding Agent, Error Recovery, Task System, Background Tasks, Cron Scheduler, Agent Teams]
main_category: "Paper Reading"
sub_category: "Agent · learn-claude-code"
discipline: "Agent"
course: "learn-claude-code"
material_type: "Source Code Study"
description: "Reviewing learn-claude-code s11-s15: Error Recovery, Task System, Background Tasks, Cron Scheduler, and Agent Teams."
lang: en
---

## 0. Overview

### The Storyline of s11-s15

These 5 lessons continue upgrading the earlier single-Agent loop into a system that is more reliable, better at running long term, and more capable of collaboration:

- s11: Error Recovery, so that when an LLM/API call fails it can be recovered by category instead of crashing outright.
- s12: Task System, which breaks a large goal into a persistable task graph with dependencies that can be claimed.
- s13: Background Tasks, which move slow tools onto background threads so the main Agent keeps working.
- s14: Cron Scheduler, which lets the Agent generate work automatically on a schedule.
- s15: Agent Teams, which lets a Lead Agent create teammates that collaborate through a mailbox.

### The Single Most Important Sentence

The model can only see `messages`. So tool results, background notifications, scheduled tasks, and teammate messages must all eventually become part of `messages` before the model can continue reasoning.

### A One-Line Memory Thread

```text
s11: how to recover when a call fails
-> s12: how to persistently manage a large task
-> s13: how to keep slow tools from blocking the main loop
-> s14: how to automatically trigger tasks at future points in time
-> s15: how multiple Agents collaborate
```

## 1. s11 Error Recovery

### What Problem It Solves

What s11 solves is:

```text
When the Agent hits an API error, truncated output, or an over-long context, it should not just crash.
```

The core change of s11 relative to s10 is:

```text
s10: call the model normally
s11: wrap a layer of recovery logic around the model call
```

### Three Recovery Paths

#### 1. max_tokens: The Output Was Truncated

Trigger condition:

```python
response.stop_reason == "max_tokens"
```

Meaning:

```text
The model did not fail; it ran out of output budget and had not finished speaking.
```

Recovery strategy:

```text
First max_tokens occurrence:
    do not append the truncated output to messages
    raise max_tokens from 8000 to 64000
    reissue the request with the same messages

If 64000 is still not enough:
    save the truncated output
    inject a continuation prompt
    let the model continue from the breakpoint
```

Why not append the truncated output the first time?

```text
Because the first time it is just that the answer sheet was too small.
The cleanest approach is to leave the original request unchanged and regenerate on a bigger answer sheet.
```

#### 2. prompt_too_long: The Input Context Is Too Long

Meaning:

```text
messages + system + tools are too long; the model's context cannot hold them.
```

Recovery strategy:

```text
reactive_compact(messages)
compact messages
then retry
```

Why can't you just wait a while and try again?

```text
Because the request content itself is too long, and waiting will not make it shorter.
You must modify the request content, that is, compact messages.
```

#### 3. 429 / 529: Transient Service Errors

429:

```text
Too Many Requests / Rate Limit
Requests are too frequent and are being throttled.
```

529:

```text
Overloaded
The server is overloaded and temporarily cannot handle the request.
```

Recovery strategy:

```text
with_retry()
exponential backoff + random jitter
wait, then retry the same request
```

Why don't 429/529 compact messages?

```text
Because there is nothing wrong with messages itself.
The problem is that the server is temporarily busy, and it may be fine after a wait.
```

### The Core s11 Comparison

| Error | Cause | How It Is Handled |
|---|---|---|
| `max_tokens` | Not enough output budget | First raise max_tokens, then continue writing |
| `prompt_too_long` | The input context is too long | Compact `messages` |
| `429` | Requests are too frequent | Wait and retry |
| `529` | The server is overloaded | Wait and retry, and switch to a backup model if necessary |

### The s11 Memory Line

```text
max_tokens: the answer sheet is too small, get a bigger one.
prompt_too_long: the input is too long, compact messages.
429/529: the service is temporarily not accepting, wait a bit.
```

## 2. s12 Task System

### What Problem It Solves

The `todo_write` of s05 is a temporary plan within the current session, suited to managing:

```text
which steps I am going to do right now
```

The Task System of s12 is a long-term task system, suited to managing:

```text
what tasks a large project breaks down into
which tasks depend on which tasks
which task is currently in progress
how to restore progress after the program restarts
```

### The Core Data Structure: Task

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

Field meanings:

- `id`: the task's unique ID.
- `subject`: the task title.
- `description`: the task's detailed description.
- `status`: `pending` / `in_progress` / `completed`.
- `owner`: who has claimed this task.
- `blockedBy`: which prerequisite tasks are blocking the current task.

### What blockedBy Is

`blockedBy` means:

```text
Which tasks I depend on; I cannot start until those tasks are complete.
```

Example:

```text
task_1: design the database tables
task_2: write the API, blockedBy=["task_1"]
task_3: write tests, blockedBy=["task_2"]
```

which means:

```text
task_2 can only start once task_1 is completed.
task_3 can only start once task_2 is completed.
```

### What claim_task Is

`claim_task` means:

```text
claim the task / start taking on this task
```

State change:

```text
pending -> in_progress
```

At the same time it sets:

```text
owner = "agent"
```

### Why can_start Is Required Before claim_task

Because you cannot skip dependencies.

`can_start(task_id)` checks:

```text
whether all tasks in blockedBy are completed
```

If the prerequisite tasks are not finished, the current task cannot be claimed.

### The Difference Between the Task System and todo_write

| Comparison | todo_write | Task System |
|---|---|---|
| Location | s05 | s12 |
| Purpose | A plan inside the current conversation | A long-term project task graph |
| Storage | Memory / messages | `.tasks/*.json` |
| Cross-session? | Not suitable | Yes |
| Dependencies? | No | Yes, via `blockedBy` |
| Has an owner? | No | Yes |
| Good for | How to do the current task | How to move a large project forward |

### The s12 Memory Line

```text
todo_write is a sticky note; the Task System is a project management board.
```

## 3. s13 Background Tasks

### What Problem It Solves

What s13 solves is:

```text
Slow tools should not block the Agent's main loop.
```

For example:

```text
npm install
pytest
docker build
```

These commands may run for a long time. If executed synchronously, the Agent can only wait for the command to return and cannot do anything else.

### What Runs in the Background

What runs in the background is:

```text
a background thread spawned by Python
```

The code pattern:

```python
threading.Thread(target=worker, daemon=True).start()
```

What actually executes inside the background thread:

```python
result = execute_tool(block)
```

If the tool is bash, then the background thread is running the bash command.

Note:

```text
It is not the model running in the background.
It is not the Agent's brain running in the background.
It is a Python thread running the slow tool in the background.
```

### What the Agent's Main Loop Is Doing

The Agent's main loop does not have to wait for the slow command to finish. It immediately returns a placeholder `tool_result`:

```text
Background task bg_0001 started.
Result will be available when complete.
```

The model can then continue requesting other tools, such as:

```text
read_file
list_tasks
write_file
```

### Why a tool_result Must Be Returned Immediately After Starting a Background Task

Because the Messages API rule is:

```text
Every tool_use must have a corresponding tool_result.
```

Even though the background task is not finished, this tool request must be "answered" first.

So what is returned is a placeholder result:

```text
This background task has started; the final result will be reported later.
```

### Why task_notification Is Used on Completion

At startup a result was already returned once:

```text
tool_use_id: abc123
tool_result: Background task bg_0001 started
```

That `tool_use` and `tool_result` pairing is already complete.

After the background work finishes, you cannot return a second `tool_result` with the same `tool_use_id`, or it would become:

```text
one tool_use corresponding to two tool_results
```

So completion uses an independent notification:

```xml
<task_notification>
  <task_id>bg_0001</task_id>
  <status>completed</status>
  <summary>...</summary>
</task_notification>
```

### The s13 Memory Line

```text
On start, return a tool_result to complete the tool_use pairing;
on completion, return a task_notification to report the async event's result.
```

## 4. s14 Cron Scheduler

### What Problem It Solves

s13 solved:

```text
There is a slow operation right now; run it in the background.
```

What s14 solves is:

```text
At some future time, automatically trigger the Agent to do something.
```

For example:

```text
run the tests at 9 a.m. every day
check CI every 5 minutes
remind me tomorrow to continue a certain task
```

### What Cron Is

cron is a kind of scheduling expression.

s14 uses the 5-field cron form:

```text
minute hour day-of-month month day-of-week
```

Examples:

```text
* * * * *        every minute
0 9 * * *        9:00 every day
*/5 * * * *      every 5 minutes
0 9 * * 1-5      9:00 Monday through Friday
```

### What CronJob Is

```python
@dataclass
class CronJob:
    id: str
    cron: str
    prompt: str
    recurring: bool
    durable: bool
```

Meanings:

- `id`: the scheduled job's ID.
- `cron`: when it fires.
- `prompt`: the message to hand to the Agent once it fires.
- `recurring`: whether it repeats.
- `durable`: whether it is saved to disk.

### What durable Is

`durable=True` means:

```text
The job definition is written into .scheduled_tasks.json
```

It can be reloaded after the program restarts.

But it does not mean:

```text
The job keeps executing automatically after the Python program is shut down.
```

The reason:

```text
What actually checks the time is the Python daemon scheduler thread.
Once the program shuts down, the thread stops too.
```

So:

```text
durable = the job definition is saved across restarts
it does not = it still runs automatically after the program is closed
```

### What daemon Is

`daemon=True` means:

```text
This is a background helper thread.
When the main program exits, it will not prevent the exit and will end along with it.
```

### The Five-Layer Core Framework of s14

This is the most important structure in s14:

```text
CronJob:
describes the job itself: when it fires and what to say once it fires

Scheduler:
only watches the clock and detects jobs that are due

cron_queue:
stores jobs that are "already due" but "not yet handled"

Queue processor:
hands jobs to the Agent when the Agent is idle

agent_loop:
turns jobs into messages so the model can handle them
```

### Why Not Let the Scheduler Call the Model Directly

Because the Agent may be running.

If `cron_scheduler_loop` called the model directly, you could get:

```text
a user-triggered agent_loop is running
the cron thread starts another agent_loop
two places modify messages at the same time
two places call tools at the same time
two places write files at the same time
```

So you need:

```text
cron_queue stores it first
the Queue processor waits for the Agent to be idle
agent_loop then consumes it
```

This is a producer-queue-consumer design:

```text
the Scheduler produces events
cron_queue buffers events
the Agent consumes events at an appropriate time
```

### The s14 Memory Line

```text
The scheduler only produces events; the Agent consumes events at an appropriate time.
```

## 5. s15 Agent Teams

### What Problem It Solves

What s15 solves is:

```text
When one Agent works on a large task, its context and attention are insufficient, so multiple Agents need to collaborate.
```

s15 introduces:

```text
Lead Agent + teammate Agents + MessageBus
```

### What spawn Is

`spawn` means:

```text
create and start.
```

In s15:

```text
The Lead Agent spawns a teammate
```

means:

```text
The main Agent creates and starts a teammate Agent.
```

The concrete code is:

```python
spawn_teammate_thread(name, role, prompt)
```

Python then starts a background thread in which the teammate runs its own agent loop.

### The Difference Between s15 and the s06 subagent

| Comparison | s06 Subagent | s15 Teammate |
|---|---|---|
| Lifecycle | One-shot | Multi-round, up to 10 rounds in the teaching version |
| Communication | Returns only the final summary | Asynchronous communication via mailbox |
| Identity | Temporary | Has a name / role |
| messages | The sub-agent's temporary messages | The teammate's own messages |
| Good for | Temporary outsourcing | Multi-Agent collaboration |

The shortest mnemonic:

```text
subagent = temporary outsourcing
teammate = a teammate with a name
```

### What MessageBus Is

`MessageBus` is a file-based message bus.

Each Agent has a mailbox file:

```text
.mailboxes/lead.jsonl
.mailboxes/alice.jsonl
.mailboxes/bob.jsonl
```

Sending a message:

```text
append one line of JSON to the recipient's .jsonl file
```

Reading messages:

```text
read your own inbox file, then delete the file after reading
```

This is called a consuming mailbox:

```text
A message is consumed once it is read.
```

### Why a teammate Has Its Own messages

Because a teammate is an independent Agent.

It needs its own:

```text
task goal
tool call records
file read results
bash output
inbox messages
intermediate context
```

If all teammates shared the Lead's `messages`, it would cause:

```text
context confusion
overly long messages
hard-to-control ordering when running in parallel
unclear responsibilities
```

So:

```text
the Lead has the Lead's messages
alice has alice's messages
bob has bob's messages
```

They exchange only the necessary information through the `MessageBus`.

### Why Teammate Messages Must Be Injected into the Lead's messages

Because the model can only see `messages`.

If alice sends a message:

```text
schema.sql created
```

but Python only stores it in:

```text
.mailboxes/lead.jsonl
```

then the Lead model does not know about it.

So Python must read the inbox and then append it to the Lead's `history/messages`:

```python
history.append({
    "role": "user",
    "content": "[Inbox]\nFrom alice: schema.sql created"
})
```

Only then, on the next model call, can the Lead see the teammate's result.

### The s15 Memory Line

```text
Each Agent keeps its context isolated and shares only the necessary results through the mailbox.
```

## 6. A Master Table of Easily Confused Points

| Easily Confused Concepts | The Difference |
|---|---|
| `max_tokens` vs `prompt_too_long` | The former is not enough output, the latter is too much input |
| `429` vs `529` | 429 means your requests are too frequent, 529 means the server is overloaded |
| `todo_write` vs Task System | The former is the current plan, the latter is a persistent task graph |
| `blockedBy` vs `owner` | `blockedBy` is dependency, `owner` is who claimed it |
| background task vs cron job | background starts running in the background now, cron fires at a future due time |
| `tool_result` vs `task_notification` | The former answers a specific tool_use, the latter reports the completion of an async event |
| durable vs daemon | durable means the job definition is saved to disk, daemon means a background thread that exits with the main program |
| subagent vs teammate | A subagent returns a summary once, a teammate can collaborate over multiple rounds via mailbox |
| mailbox vs messages | A mailbox is a file relay station, messages is the context the model can actually see |

## 7. A Unified Mental Model

From s11 to s15, everything actually revolves around one core idea:

```text
Python is responsible for maintaining the external world and program state;
the model only sees that state through messages;
so every important event must ultimately enter messages.
```

Concretely:

```text
s11:
after error recovery, keep calling the model with messages

s12:
task state is written into .tasks, but the model must see it through tool results

s13:
after background completion, inject a task_notification into messages

s14:
once cron is due, inject the prompt into messages

s15:
after a teammate sends a message, inject the inbox into the Lead's messages
```

So the most important review sentence is:

```text
The model does not perceive files, threads, time, or mailboxes directly; Python must convert those events into messages.
```

## 8. Second-Pass Question List

1. Why don't you immediately append the truncated output the first time you hit `max_tokens`?
2. Why do `429/529` use `with_retry` while `prompt_too_long` requires compacting `messages`?
3. What is the biggest difference between `todo_write` and the s12 Task System?
4. What does `blockedBy` mean? Why is `can_start` required before `claim_task`?
5. Why must a placeholder `tool_result` be returned immediately after a background task starts?
6. Why is `<task_notification>` used on background completion instead of reusing the same `tool_use_id`?
7. What is the responsibility of each layer in `CronJob -> Scheduler -> cron_queue -> Queue processor -> agent_loop`?
8. Does `durable=True` mean cron still runs automatically after the Python program is shut down? Why?
9. What is the difference between the s06 subagent and the s15 teammate?
10. Why does a teammate need its own `messages`?
11. Why must teammate messages be injected into the Lead's `history/messages`?
12. Why do we say "the model can only see external events through messages"?