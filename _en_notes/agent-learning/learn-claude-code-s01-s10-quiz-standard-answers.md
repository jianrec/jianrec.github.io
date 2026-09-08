---
title: "learn-claude-code s01-s10: Quiz Answer Key Review Notes"
date: 2026-06-27 20:26:04 +0800
tags: [Agent, Claude Code, Coding Agent, Quiz, Tool Use, Hooks, Memory, Skill Loading, Context Compact]
main_category: "Paper Reading"
sub_category: "Agent · learn-claude-code"
discipline: "Agent"
course: "learn-claude-code"
material_type: "Quiz Notes"
description: "A compilation of the quiz answer key for learn-claude-code s01-s10, covering the agent loop, multiple tools, permissions, hooks, todos, subagents, skill loading, context compaction, memory, and dynamic system prompts."
lang: en
ref: "learn-claude-code-s01-s10-quiz-standard-answers"
---

## 0. Overview

### The Through-Line of s01-s10

These 10 lessons build up an increasingly complete Agent system step by step:

- s01: The basic Agent Loop. The model decides whether to issue a tool call; Python actually executes the tool.
- s02: A multi-tool system. Adds file read/write, edit, and glob, dispatching via a handler dictionary.
- s03: The permission system. Blocks dangerous tool calls at the Python layer.
- s04: Hooks. Moves extension logic such as logging, checks, and automatic handling out of the main loop.
- s05: TodoWrite. Lets the model manage the plan and status of complex tasks.
- s06: Subagent. Hands complex tasks to a sub-agent; the main Agent only receives a summarized result.
- s07: Skill Loading. Extends the model's capabilities via a lightweight catalog plus on-demand loading.
- s08: Context Compact. Compresses the current `messages` to prevent the context from growing too long.
- s09: Memory. Persists long-term valuable information to disk for reuse across sessions.
- s10: System Prompt. Splits the system prompt into sections and assembles them dynamically based on context.

### The Single Most Important Sentence

The model is responsible for "understanding, reasoning, and deciding whether to request a tool";

Python is responsible for "maintaining `messages`, sending API requests, executing tools, checking permissions, triggering hooks, saving state, and returning results to the model."

### One Through-Line to Memorize

```text
s01 basic loop
-> s02 multiple tools
-> s03 permission checks
-> s04 hook decoupling
-> s05 todo planning
-> s06 subagent division of labor
-> s07 skill capability loading
-> s08 compact the current context
-> s09 memory for the long term
-> s10 dynamic system prompt assembly
```

## 1. What Is the Complete Flow of the s01 Agent Loop?

User input is first placed into `messages`. Python calls `client.messages.create`, sending `messages`, `system`, and the `tools schema` to the model together.

The model returns either plain text or a `tool_use`. If it returns `tool_use`, Python looks up the actual function by tool name, for example `run_bash`, and then executes the tool.

Once the tool finishes, Python wraps the result as a `tool_result`, puts it back into `messages`, and calls the model again so it can continue answering based on the tool result.

Key takeaway: the model decides whether to request a tool; Python is what actually executes it.

## 2. Why Do We Say the Model Receives a tools schema Rather than the Python Functions Themselves?

The `tools schema` is the tool manual written for the model. It tells the model:

- What the tool is called
- What the tool can do
- What parameters the tool needs
- What types those parameters are

But it does not contain the Python function code. The model cannot see the implementation of `def run_bash(...)`, so it cannot execute the function itself; it can only return a tool call request.

## 3. What Is the Difference Between the TOOLS schema and TOOL_HANDLERS?

The `TOOLS schema` is the tool menu shown to the model.

`TOOL_HANDLERS` is the tool routing table used by Python.

The model returns a structured request along the lines of "I want to call `read_file` with parameters `{"path": "a.txt"}`". Python then uses the tool name to find the actual function in `TOOL_HANDLERS` and executes it:

```python
handler = TOOL_HANDLERS.get(block.name)
output = handler(**block.input)
```

## 4. Why Can't s03 Rely on the system prompt Alone for Permission Control?

A prompt only persuades the model to follow the rules; the Python permission check is the actual gate.

The model might request a dangerous command, for example:

```bash
rm -rf /
```

Without a `DENY_LIST` or `check_permission` at the Python layer, the program might genuinely perform a dangerous operation.

So the security boundary must live in the program. The model may propose a tool call request, but Python must decide whether to allow it before executing.

## 5. What Problem Do hooks Solve in the s04 Agent Loop?

`hook` moves "the extra things that need to happen during the loop" out of the `agent_loop` main flow.

Without hooks, every time you want to add a feature — logging before a tool call, automatic processing after a tool call, saving results on stop — you would have to modify `agent_loop`.

With hooks, the main loop only needs to fire at fixed points:

```python
trigger_hooks("PreToolUse", ...)
trigger_hooks("PostToolUse", ...)
trigger_hooks("Stop", ...)
```

What logic actually runs is decided by the hook functions that were registered.

In one sentence: a hook is a slot reserved at key points in the main loop for external functionality to plug into.

## 6. What Is the Biggest Difference Between a hook and a tool?

A `tool` is a capability the model can see and request, such as `bash`, `read_file`, or `todo_write`.

A `hook` is an extension point triggered automatically inside Python; the model usually cannot see it and never invokes it directly.

In one sentence:

```text
tool is for the model to use; hook is for the program's internal lifecycle.
```

## 7. Why Does s05 Introduce todo_write?

`todo_write` exists to let the model manage the plan and status of complex tasks.

Its purpose is not to read files, write files, or execute commands, but to:

- Break a complex task into to-do items
- Record the status of each step
- Let the user and the program see the current progress
- Prevent the model from forgetting the task partway through

In one sentence:

```text
Ordinary tools do the work; todo_write manages the plan for doing the work.
```

## 8. Do a Sub-Agent's Detailed Tool Calls Enter the Main Agent's messages?

No.

If the main Agent calls the `task` tool and the sub-agent internally reads 10 files and runs bash 5 times, those details exist only in the sub-agent's own temporary loop.

The main Agent's `messages` only stores the final summary string returned by the `task` tool.

This keeps the main Agent's context cleaner and prevents it from being blown out by the sub-agent's many intermediate steps.

## 9. How Does SKILL.md Become a Skill Catalog the Model Can See?

The full chain is:

```text
SKILL.md file
-> _parse_frontmatter() extracts name / description
-> _scan_skills() scans and registers all skills
-> stored into SKILL_REGISTRY
-> list_skills() turns the dictionary into a plain-text catalog
-> build_system() places the catalog into the system prompt
-> client.messages.create(system=SYSTEM) sends it to the model
```

What the model initially sees is a lightweight skill catalog, not every complete `SKILL.md`.

If the model needs a particular skill, it then calls `load_skill(name)`, and only then does Python return the full skill content to the model.

## 10. Why Not Just Stuff All SKILL.md Files into the system prompt?

You could, but it is costly and the results may be worse.

The main problems are:

- The context becomes too large
- Tokens are wasted
- The current task may need only one skill
- Too many irrelevant skills interfere with the model's judgment
- The system prompt becomes hard to maintain

So s07 uses progressive disclosure:

```text
Give the model a lightweight skill catalog first
The model decides which skill it needs
Then use load_skill to read the full SKILL.md
```

## 11. Why Does s08 Introduce context compact?

s08 exists to keep `messages` from piling up and tool results from growing ever longer, eventually exceeding the model's context limit and causing the API to raise `prompt_too_long`.

It mainly handles:

- Large tool results
- Old `tool_result` entries
- Intermediate history messages
- Overly long conversation context

The goal is to squeeze the current context down into a range the model can accept while preserving as much useful information as possible.

## 12. Why Does s08 Handle Large tool_results First and Summarize History Only Later?

The principle is: handle things cheaply when you can, and only fall back to expensive, potentially lossy summarization when that is not enough.

The compaction layers in s08 are roughly:

```text
1. persist_large_output
   Write especially large tool results to a file, keeping only the path and a preview

2. micro_compact
   Replace old, long tool_results with placeholder information

3. snip_compact
   Trim out a middle portion of the history, keeping only the beginning and the most recent rounds

4. compact_history
   Have the model summarize the old history and replace the original messages with the summary
```

The reason not to summarize right away is that summarization loses detail, requires an extra model call, and can also get the summary wrong.

## 13. What Is the Difference Between s08 compact and s09 memory?

`s08 compact` deals with the current conversation context being too long.

`s09 memory` deals with long-term memory across sessions.

More concretely:

```text
compact deals with messages
memory deals with .memory/*.md files
```

```text
compact's goal: make the current context shorter
memory's goal: persist user preferences, project facts, and long-term constraints
```

## 14. Why Does s09 Save pre_compress?

Because the compressed `messages` may already have lost detail.

Extracting memory directly from the compressed content risks missing important information such as user preferences, project facts, weak spots, and long-term constraints.

So s09 temporarily saves a more complete copy of the history before compaction:

```python
pre_compress = messages.copy()
```

and then uses this more complete `pre_compress` to extract long-term memory.

Note: this does not permanently stuff the entire overlong conversation back into the main context; it is only used temporarily within the current Python flow.

## 15. What Is the Biggest Change in s10 Compared with Earlier Versions?

s10 no longer uses a single hard-coded `SYSTEM` string. Instead it assembles the system prompt dynamically based on the current `context`.

It splits the prompt into multiple sections, for example:

- `identity`
- `tools`
- `workspace`
- `memory`

Then it decides, based on the current context, which sections go into the final system prompt.

In one sentence:

```text
system prompt = the prompt assembled dynamically according to context.
```

## 16. If .memory/MEMORY.md Does Not Exist, Will the memory Section Enter the system prompt?

No.

Because the logic in s10 is roughly:

```python
memories = context.get("memories")
if memories:
    sections.append(memory_section)
```

In Python, an empty string `""` and `None` are both treated as False.

If `.memory/MEMORY.md` does not exist, `update_context()` cannot read any memory, so `context["memories"]` will be empty, `if memories` will not hold, and the memory section will not be added to the final system prompt.

## 17. How Does s01 Evolve into s10, Step by Step?

The overall evolution path is:

```text
s01: basic Agent Loop and the bash tool
s02: multi-tool system and TOOL_HANDLERS dispatch
s03: permission checks
s04: hooks decouple extension logic
s05: todo_write plan management
s06: task / subagent division of labor
s07: skill loading, a lightweight catalog + on-demand loading of full skills
s08: context compact, compressing the current messages
s09: memory, persisting long-term memory across sessions
s10: dynamic system prompt, assembling the prompt from context
```

The through-line is:

```text
From "the model requests a tool and Python executes it"
gradually expanding into
an Agent system that is "secure, extensible, plan-driven, capable of division of labor, able to load capabilities, compact context, retain long-term memory, and dynamically assemble prompts."
```

## 18. What Is the Difference Between s07 skill loading and s09 memory?

In one sentence:

```text
skill = what I know how to do
memory = what I remember
```

`skill loading` loads capability descriptions, telling the model how a certain kind of task should be done and what rules and steps apply.

`memory` loads information remembered from the past, telling the model about user preferences, project facts, long-term constraints, and so on.

## 19. To Automatically Log After Every Tool Call, Should You Use a tool, a hook, memory, or the system prompt?

You should use a `hook`, specifically hanging it on the `PostToolUse hook`.

Because this feature is not a capability the model should invoke, nor long-term memory, nor a prompt rule. It is something Python performs automatically after each tool call finishes.

The flow is:

```text
Tool execution finishes
-> trigger_hooks("PostToolUse", tool_name, result)
-> the logging hook automatically records the tool name and result length
```

This is exactly the classic use of a hook: inserting extra program logic at fixed points in the agent loop without modifying the main loop.