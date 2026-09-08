---
title: "learn-claude-code s06-s10: Context and Capability Review Notes"
date: 2026-06-27 20:31:43 +0800
tags: [Agent, Claude Code, Coding Agent, Subagent, Skill Loading, Context Compact, Memory, System Prompt]
main_category: "Paper Reading"
sub_category: "Agent · learn-claude-code"
discipline: "Agent"
course: "learn-claude-code"
material_type: "Source Code Study"
description: "Reviewing learn-claude-code s06-s10: sub-agent context isolation, on-demand skill loading, context compaction, long-term memory, and dynamic system prompts."
lang: en
ref: "learn-claude-code-s06-s10-context-capability-review"
---

Created: 2026-06-26  
Scope: `s06_subagent` through `s10_system_prompt`  
Goal: review how the Agent harness evolves in context isolation, knowledge loading, context compaction, long-term memory, and system prompt assembly.

---

## 0. Memorize These Five Sentences First

1. `s06`: a sub-agent completes a subtask on its own fresh `messages[]`, and the main Agent only receives the summary.
2. `s07`: Skill Loading first gives the model a skill catalog, then loads the full `SKILL.md` with `load_skill` when needed.
3. `s08`: Context Compact cleans up `messages` before every LLM call to keep the context window from blowing up.
4. `s09`: Memory writes long-term useful information into `.memory/`, preserved across compaction and across sessions.
5. `s10`: the System Prompt is no longer hardcoded; it is assembled at runtime from the current `context`.

---

## 1. The Overall Storyline of s06-s10

`s01-s05` addresses whether the Agent can act at all:

```text
tool_use -> Python handler -> tool_result -> loop
```

`s06-s10` addresses how the Agent can act over the long run, professionally and stably:

```text
s06: the complex process is too noisy -> isolate it with a sub-agent
s07: there is too much domain knowledge -> load skills on demand
s08: the historical context is too long -> shrink it with compaction
s09: compaction loses important information -> preserve it with memory
s10: the system prompt is too messy -> assemble it from sections at runtime
```

You can see these five sections as different facets of the same problem:

```text
Context is a finite resource.

subagent: keep the subtask process from polluting the main context
skill: keep irrelevant knowledge from entering the context ahead of time
compact: keep the message history from growing without bound
memory: keep important information from disappearing because of compaction
system prompt: keep system instructions from becoming one unmaintainable blob of text
```

---

## 2. s06: Subagent, Context Isolation

File: `s06_subagent/code.py`

### 2.1 The Problem s06 Solves

A complex task generates a lot of intermediate process. For example:

```text
read 20 files
run 10 commands
analyze the call chain
in the end only one sentence of conclusion is needed
```

If all of that intermediate process stays in the main Agent's `messages`, the main context becomes very noisy.  
So s06 adds a `task` tool that lets the main Agent dispatch a sub-agent to handle a side task.

### 2.2 task Is an Ordinary tool

Code location: `s06_subagent/code.py:252`

```python
TOOLS.append({
    "name": "task",
    "description": "Launch a subagent to handle a complex subtask. Returns only the final conclusion.",
    "input_schema": {"type": "object", "properties": {"description": {"type": "string"}}, "required": ["description"]},
})
TOOL_HANDLERS["task"] = spawn_subagent
```

This is exactly the same as the earlier tool mechanism:

```text
The model sees the task schema
-> the model returns tool_use: task
-> Python looks up TOOL_HANDLERS["task"]
-> executes spawn_subagent(description)
```

So there is no magic in `task`. It is just a tool whose handler happens to be special.

### 2.3 The Sub-agent Has Its Own system prompt

Code location: `s06_subagent/code.py:58`

```python
SUB_SYSTEM = (
    f"You are a coding agent at {WORKDIR}. "
    "Complete the task you were given, then return a concise summary. "
    "Do not delegate further."
)
```

This shows the sub-agent's goal is narrower:

```text
Complete the task you were given
Return a concise summary
Do not delegate further
```

### 2.4 The Sub-agent Has Its Own Tool Set

Code location: `s06_subagent/code.py:182`

```python
SUB_TOOLS = [
    bash,
    read_file,
    write_file,
    edit_file,
    glob,
]
```

Note: `SUB_TOOLS` does not include `task`.

The purpose:

```text
prevent main Agent -> sub-agent -> sub-sub-agent -> infinite recursion
```

### 2.5 spawn_subagent Is the Core

Code location: `s06_subagent/code.py:207`

```python
def spawn_subagent(description: str) -> str:
    print("[Subagent spawned]")
    messages = [{"role": "user", "content": description}]
```

The most critical line is this one:

```python
messages = [{"role": "user", "content": description}]
```

What it creates is the sub-agent's own local `messages`, not the main Agent's `history`.

So:

```text
Main Agent messages: the user's original task + the main flow history
Sub-agent messages: the subtask description + the sub-agent's internal process
```

That is context isolation.

### 2.6 The Sub-agent Runs Its Own loop

Code location: `s06_subagent/code.py:212`

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

The sub-agent essentially has a miniature agent loop as well:

```text
call the LLM
-> if tool_use
-> execute SUB_HANDLERS
-> put the tool_result back into the sub messages
-> continue
```

`for _ in range(30)` is a safety cap that prevents the sub-agent from looping forever.

### 2.7 Return Only the Summary, Not the Process

Code location: `s06_subagent/code.py:249`

```python
return result
```

In the end the main Agent gets only a string:

```text
the sub-agent's final summary
```

It will not receive the detailed process of the sub-agent's 10 file reads and 5 bash calls.

But note:

```text
The sub-agent's messages do not enter the main Agent.
The sub-agent's real side effects are kept.
```

For example, if the sub-agent wrote a file, that file really does exist.

### 2.8 The s06 Review Mnemonic

```text
task is the entry point
spawn_subagent is the execution
fresh messages are the isolation
SUB_TOOLS without task prevents recursion
return summary reduces context
```

---

## 3. s07: Skill Loading, Knowledge Loaded On Demand

File: `s07_skill_loading/code.py`

### 3.1 The Problem s07 Solves

An Agent needs domain knowledge, for example:

```text
code review conventions
PDF processing workflow
how to write an MCP server
agent architecture guide
```

Stuffing all of it into `SYSTEM` up front wastes tokens and makes the context noisy.

The s07 design has two layers:

```text
Layer one: the skill catalog stays resident in the system prompt
Layer two: the full SKILL.md is loaded on demand via load_skill
```

### 3.2 The skills/ Directory

Code location: `s07_skill_loading/code.py:47`

```python
SKILLS_DIR = WORKDIR / "skills"
```

The skill file structure in the project:

```text
skills/
  agent-builder/SKILL.md
  code-review/SKILL.md
  mcp-builder/SKILL.md
  pdf/SKILL.md
```

Each `SKILL.md` has frontmatter at the top:

```yaml
---
name: code-review
description: Perform thorough code reviews...
---
```

### 3.3 _parse_frontmatter Parses the Short Information

Code location: `s07_skill_loading/code.py:53`

```python
def _parse_frontmatter(text: str) -> tuple[dict, str]:
    if not text.startswith("---"):
        return {}, text
    parts = text.split("---", 2)
    meta = yaml.safe_load(parts[1]) or {}
    return meta, parts[2].strip()
```

It extracts from the top of `SKILL.md`:

```text
name
description
```

At this step nothing has been given to the LLM yet; it is just parsing inside Python.

### 3.4 _scan_skills Builds the Registry

Code location: `s07_skill_loading/code.py:69`

```python
def _scan_skills():
    ...
    raw = manifest.read_text()
    meta, body = _parse_frontmatter(raw)
    name = meta.get("name", d.name)
    desc = meta.get("description", ...)
    SKILL_REGISTRY[name] = {"name": name, "description": desc, "content": raw}
```

`SKILL_REGISTRY` is a skill database in Python memory.

It looks roughly like this:

```python
{
    "code-review": {
        "name": "code-review",
        "description": "Perform thorough code reviews...",
        "content": "the full SKILL.md content"
    }
}
```

Note: the LLM cannot read Python variables directly. They must be turned into text and sent to it via `system` / `messages` / `tool_result`.

### 3.5 list_skills Generates a Lightweight Catalog

Code location: `s07_skill_loading/code.py:86`

```python
def list_skills() -> str:
    if not SKILL_REGISTRY:
        return "(no skills found)"
    return "\n".join(f"- **{s['name']}**: {s['description']}" for s in SKILL_REGISTRY.values())
```

This step turns the Python dictionary into a plain text menu:

```text
- **agent-builder**: Design and build AI agents...
- **code-review**: Perform thorough code reviews...
- **mcp-builder**: Build MCP servers...
- **pdf**: Process PDF files...
```

Humans can read this text, and so can the LLM.

### 3.6 build_system Sends the Catalog to the LLM

Code location: `s07_skill_loading/code.py:93`

```python
def build_system() -> str:
    catalog = list_skills()
    return (
        f"You are a coding agent at {WORKDIR}. "
        f"Skills available:\n{catalog}\n"
        "Use load_skill to get full details when needed."
    )
```

It is finally sent to the LLM through:

```python
client.messages.create(..., system=SYSTEM, ...)
```

The full chain:

```text
SKILL.md file
-> _parse_frontmatter()
-> _scan_skills()
-> SKILL_REGISTRY
-> list_skills()
-> build_system()
-> client.messages.create(system=SYSTEM)
-> LLM
```

### 3.7 load_skill Loads the Full Content

Code location: `s07_skill_loading/code.py:269`

```python
def load_skill(name: str) -> str:
    skill = SKILL_REGISTRY.get(name)
    if not skill:
        return f"Skill not found: {name}"
    return skill["content"]
```

After seeing the catalog, if the model decides it needs a certain skill, it can call:

```python
load_skill({"name": "code-review"})
```

Python returns the full `SKILL.md`, which enters `messages` as a `tool_result`.

### 3.8 The s07 Review Mnemonic

```text
skill files live in skills/
frontmatter yields name/description
SKILL_REGISTRY lives in Python memory
list_skills turns it into a text menu
build_system sends the catalog
load_skill sends the full text
```

---

## 4. s08: Context Compact, Context Compaction

File: `s08_context_compact/code.py`

### 4.1 The Problem s08 Solves

`messages` keeps growing:

```text
user prompt
assistant tool_use
user tool_result
assistant tool_use
user tool_result
...
```

Reading files, running commands, and loading skills all add context.  
Without compaction, the API will eventually report:

```text
prompt_too_long
too many tokens
```

The s08 principle:

```text
Run the cheap compaction first, the expensive compaction last.
```

### 4.2 Three Constants

Code location: `s08_context_compact/code.py:265`

```python
CONTEXT_LIMIT = 50000
KEEP_RECENT = 3
PERSIST_THRESHOLD = 30000
```

Meanings:

```text
CONTEXT_LIMIT: exceeding this estimated size triggers LLM summarization
KEEP_RECENT: the most recent 3 tool_results are kept in full
PERSIST_THRESHOLD: a single large output longer than this is written to disk
```

The teaching version estimates size with `len(str(messages))`, which is not a real token count.

### 4.3 snip_compact: Trimming the Middle Messages

Code location: `s08_context_compact/code.py:295`

```python
def snip_compact(messages, max_messages=50):
    if len(messages) <= max_messages:
        return messages
    keep_head, keep_tail = 3, max_messages - 3
    ...
    return messages[:head_end] + [{"role": "user", "content": f"[snipped {snipped} messages]"}] + messages[tail_start:]
```

What it does:

```text
keep the first few messages
keep the most recent few dozen
replace the old messages in the middle with a placeholder
```

Why can't you cut arbitrarily?

Because `assistant(tool_use)` and `user(tool_result)` form a pair, and you cannot leave half a pair behind.

### 4.4 micro_compact: Placeholders for Old Tool Results

Code location: `s08_context_compact/code.py:322`

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

What it does:

```text
the most recent 3 tool_results are kept in full
older long tool_results are replaced with a one-line placeholder
```

This is lossy compaction. The old content is gone, but the model can re-read the file or re-run the command.

### 4.5 tool_result_budget: Persisting Large Output to Disk

Code location: `s08_context_compact/code.py:332`

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

What it does:

```text
the complete large output is written to disk
messages keeps only the path + a preview
```

This is more robust than throwing it away, because the full content is still at:

```text
.task_outputs/tool-results/<tool_use_id>.txt
```

### 4.6 compact_history: LLM Summarization

Code location: `s08_context_compact/code.py:375`

```python
def compact_history(messages):
    transcript_path = write_transcript(messages)
    print(f"[transcript saved: {transcript_path}]")
    summary = summarize_history(messages)
    return [{"role": "user", "content": f"[Compacted]\n\n{summary}"}]
```

What it does:

```text
write the full history to a transcript
have the LLM summarize the current goal, key findings, changed files, remaining work, and user constraints
replace the old messages with a single summary
```

This is the most expensive layer, because it requires an extra LLM call.

### 4.7 reactive_compact: Emergency Compaction

Code location: `s08_context_compact/code.py:383`

```python
def reactive_compact(messages):
    ...
    return [{"role": "user", "content": f"[Reactive compact]\n\n{summary}"}, *messages[tail_start:]]
```

If the API still reports `prompt_too_long` when the request is made, it compacts as an emergency measure and retries once.

### 4.8 Wiring Compaction into agent_loop

Code location: `s08_context_compact/code.py:454`

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

Note the actual execution order:

```text
tool_result_budget -> snip_compact -> micro_compact -> compact_history
```

Although the explanation calls them L1/L2/L3/L4, at execution time the large output is persisted first, so that the full content is not lost when placeholders are later inserted.

### 4.9 The compact Tool

Code location: `s08_context_compact/code.py:416`

```python
{"name": "compact", "description": "Summarize earlier conversation to free context space."}
```

The model can call `compact` on its own initiative to trigger `compact_history()`.

### 4.10 The s08 Review Mnemonic

```text
messages get longer the further you run
persist large output to disk first
trim the old messages in the middle
put placeholders in for old tool_results
if it is still too big, summarize with the LLM
if the API errors, reactive compact
```

---

## 5. s09: Memory, Long-Term Memory

File: `s09_memory/code.py`

### 5.1 The Problem s09 Solves

s08's compaction lets the session continue, but compaction is lossy.  
Important information may be summarized too coarsely, or simply lost in a new session.

For example, the user says:

```text
From now on, please explain code in Chinese and give more examples.
```

This should not stay only in the current `messages`; it should be preserved long term.

The s09 approach:

```text
write important information to the .memory/ file system
use MEMORY.md as the index
load relevant memories at the start of every turn
extract new memories at the end of every turn
merge and tidy up once there are too many memories
```

### 5.2 The Memory Directory

Code location: `s09_memory/code.py:42`

```python
MEMORY_DIR = WORKDIR / ".memory"
MEMORY_INDEX = MEMORY_DIR / "MEMORY.md"
```

The directory structure:

```text
.memory/
  MEMORY.md
  user-prefers-chinese.md
  project-facts.md
  feedback-style.md
```

### 5.3 Four Kinds of Memory

Code location: `s09_memory/code.py:56`

```python
MEMORY_TYPES = ["user", "feedback", "project", "reference"]
```

Meanings:

```text
user: who the user is, user preferences
feedback: the user's feedback on how the Agent does things
project: project facts
reference: external leads or reference locations
```

### 5.4 write_memory_file: Writing a Single Memory

Code location: `s09_memory/code.py:72`

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

The format of a single memory file:

```markdown
---
name: user-prefers-chinese
description: User prefers Chinese explanations
type: user
---

User prefers Chinese explanations with concrete examples.
```

### 5.5 _rebuild_index: Rebuilding MEMORY.md

Code location: `s09_memory/code.py:84`

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

`MEMORY.md` is a memory catalog, not the full text:

```markdown
- [user-prefers-chinese](user-prefers-chinese.md) — User prefers Chinese explanations
```

### 5.6 build_system: Putting the Index into SYSTEM

Code location: `s09_memory/code.py:337`

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

This is very similar to s07:

```text
s07: the skill catalog goes into SYSTEM
s09: the memory index goes into SYSTEM
```

The index stays resident; the full text is loaded on demand.

### 5.7 select_relevant_memories: Choosing Relevant Memories

Code location: `s09_memory/code.py:132`

```python
def select_relevant_memories(messages: list, max_items: int = 5) -> list[str]:
```

It will:

```text
read the name/description of every memory file
collect the recent user messages
have the LLM pick the relevant items from the memory catalog
fall back to keyword matching on failure
pick at most 5 items
```

### 5.8 load_memories: Reading the Full Text of Relevant Memories

Code location: `s09_memory/code.py:207`

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

It returns:

```xml
<relevant_memories>
...full text of the relevant memories...
</relevant_memories>
```

### 5.9 Injecting Memory into the Current Request

Code location: `s09_memory/code.py:583`

```python
memories_content = load_memories(messages)
memory_turn = len(messages) - 1 if messages and isinstance(messages[-1].get("content"), str) else None
system = build_system()
```

Code location: `s09_memory/code.py:606`

```python
request_messages = messages
if memories_content and memory_turn is not None and memory_turn < len(messages):
    request_messages = messages.copy()
    request_messages[memory_turn] = {
        **messages[memory_turn],
        "content": memories_content + "\n\n" + messages[memory_turn]["content"],
    }
```

Note that it does not modify the original `messages` directly; it constructs `request_messages` and sends that to the LLM.

What the LLM actually sees:

```text
<relevant_memories>
...
</relevant_memories>

the user's current question
```

### 5.10 extract_memories: Extracting New Memories at the End of Each Turn

Code location: `s09_memory/code.py:222`

```python
def extract_memories(messages: list):
```

It extracts from the recent conversation:

```text
user preferences
user constraints
project facts
user feedback
```

and asks the LLM to return JSON:

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

It then calls `write_memory_file()` to write to disk.

### 5.11 pre_compress: A Snapshot Before Compaction

Code location: `s09_memory/code.py:592`

```python
pre_compress = [...]
```

Why save a pre-compaction version?

Because s08 may have compacted the details away.  
When extracting memories, you need to see a more complete original conversation.

Note:

```text
pre_compress temporarily occupies Python RAM
it is not long-term storage
it is not the main LLM context
it is released once the function returns
```

### 5.12 consolidate_memories: Tidying Up Memories

Code location: `s09_memory/code.py:287`

```python
def consolidate_memories():
    files = list_memory_files()
    if len(files) < CONSOLIDATE_THRESHOLD:
        return
```

Once there are 10 memory files, it has the LLM:

```text
merge duplicates
delete stale entries
resolve contradictions
keep important preferences
```

### 5.13 The s09 Review Mnemonic

```text
messages are short-term memory
.memory is long-term memory
MEMORY.md is the index
relevant memories are injected on demand
new memories are extracted at the end of each turn
when there are too many, consolidate
```

---

## 6. s10: System Prompt, Assembled at Runtime

File: `s10_system_prompt/code.py`

### 6.1 The Problem s10 Solves

The earlier chapters keep adding content to the system prompt:

```text
identity
tools
working directory
skill catalog
memory index
permission notes
context rules
```

If you keep using a single hardcoded `SYSTEM` string, it becomes harder and harder to maintain.

The s10 approach:

```text
split the system prompt into sections
concatenate them on demand based on the current real context
if the context has not changed, use the cache
```

### 6.2 PROMPT_SECTIONS: Defining the Available Fragments

Code location: `s10_system_prompt/code.py:42`

```python
PROMPT_SECTIONS = {
    "identity": "You are a coding agent. Act, don't explain.",
    "tools": "Available tools: bash, read_file, write_file.",
    "workspace": f"Working directory: {WORKDIR}",
    "memory": "Relevant memories are injected below when available.",
}
```

This splits one whole block of system prompt into multiple topics.

### 6.3 assemble_system_prompt: Concatenating by context

Code location: `s10_system_prompt/code.py:50`

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

Always loaded:

```text
identity
tools
workspace
```

Conditionally loaded:

```text
memory
```

If `.memory/MEMORY.md` does not exist or is empty, the `memory` section does not make it into the final prompt.

### 6.4 update_context: Reading the Real State

Code location: `s10_system_prompt/code.py:156`

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

The `context` here comes from real state:

```text
which tools are actually registered
what the current working directory is
whether MEMORY.md really exists and has content
```

It is not guessed from whether keywords appear in the user's message.

### 6.5 get_system_prompt: Caching

Code location: `s10_system_prompt/code.py:71`

```python
def get_system_prompt(context: dict) -> str:
    key = json.dumps(context, sort_keys=True, ensure_ascii=False, default=str)
    if key == _last_context_key and _last_prompt:
        return _last_prompt
    _last_context_key = key
    _last_prompt = assemble_system_prompt(context)
    return _last_prompt
```

What it does:

```text
context unchanged -> return the previous prompt directly
context changed -> reassemble the prompt
```

Why use `json.dumps`?

```text
dict/list cannot be hashed directly
Python hashes are not stable
json.dumps(sort_keys=True) produces a stable string key
```

Note: this is a local string cache, not an API prompt cache.

### 6.6 agent_loop Uses a Dynamic system

Code location: `s10_system_prompt/code.py:172`

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

It used to be:

```python
system=SYSTEM
```

Now it is:

```python
system=get_system_prompt(context)
```

Recomputed after tool execution:

Code location: `s10_system_prompt/code.py:195`

```python
context = update_context(context, messages)
system = get_system_prompt(context)
```

### 6.7 The s10 Review Mnemonic

```text
PROMPT_SECTIONS defines the materials
update_context reads the real state
assemble_system_prompt puts them together on demand
get_system_prompt does the caching
client.messages.create sends it to the LLM
```

---

## 7. A Side-by-Side Comparison of s06-s10

| Section | Problem Solved | Core Data Structure | Key Functions | End Effect |
| --- | --- | --- | --- | --- |
| s06 | The subtask process pollutes the main context | The sub-agent's local `messages[]` | `spawn_subagent()` | The main Agent receives only a summary |
| s07 | Too much domain knowledge to stuff into the prompt | `SKILL_REGISTRY` | `list_skills()`, `load_skill()` | The skill catalog stays resident, the full text is on demand |
| s08 | `messages` keeps getting longer | `messages` + transcript + tool output files | `snip_compact()`, `micro_compact()`, `compact_history()` | Long sessions do not blow the context |
| s09 | compact loses important information | `.memory/*.md`, `MEMORY.md` | `extract_memories()`, `load_memories()` | Preferences and facts are remembered long term |
| s10 | The system prompt gets messier | `PROMPT_SECTIONS`, `context` | `assemble_system_prompt()`, `get_system_prompt()` | The prompt is assembled dynamically |

---

## 8. Three Pairs of Easily Confused Concepts

### 8.1 skill vs memory

```text
skill:
- domain knowledge written in advance by a human
- lives in skills/
- examples: code-review, pdf, mcp-builder
- loaded on demand with load_skill

memory:
- long-term information the Agent accumulates from conversation
- lives in .memory/
- examples: user preferences, project facts, feedback
- extract_memories at the end of each turn
```

### 8.2 compact vs memory

```text
compact:
- solves the current messages being too long
- the goal is to reduce context volume
- details may be dropped

memory:
- solves preserving important information long term
- the goal is to remember across compaction and across sessions
- should not be dropped casually
```

In one sentence:

```text
compact is responsible for forgetting the unimportant.
memory is responsible for remembering the important.
```

### 8.3 SYSTEM vs messages vs tool_result

```text
SYSTEM:
- high-priority long-term instructions for the model
- s07 puts the skill catalog here
- s09 puts the memory index here
- s10 assembles it dynamically

messages:
- the current conversation history
- it grows, and it gets compacted

tool_result:
- the result of executing a tool
- returned to the model as the content of a user message
```

---

## 9. Imagining the Five Sections Executing Together

Suppose the user says:

```text
Please review this project, and remember to explain in Chinese from now on.
```

What might happen:

```text
1. s10 update_context()
   assembles the system prompt from tools, workspace, and memory

2. s07 puts the skill catalog into the system prompt
   the model sees the code-review skill

3. the model calls load_skill("code-review")
   the full review guide enters the tool_result

4. when the task is large, the model calls task
   s06 spawn_subagent does the subtask on independent messages

5. there are many file reads and tool outputs
   s08 compacts messages before the next LLM call

6. the user says "use Chinese from now on"
   s09 runs extract_memories at the end of the turn
   and writes .memory/user-prefers-chinese.md

7. at the start of the next conversation
   s10/s09 bring the MEMORY.md index and the relevant memories back into the context
```

That is the complete capability of `s06-s10` combined.

---

## 10. The Spoken Summary

If someone asks you: what are `s06-s10` about?

You can answer like this:

> These five sections add context and knowledge management capabilities to the Agent. s06 uses a sub-agent to isolate a complex subtask and returns only the final summary to the main Agent. s07 uses two-level skill loading so the model first sees the skill catalog and only loads the full skill document when needed. s08 compacts messages before every LLM call to keep the context from getting too long. s09 writes user preferences and project facts into `.memory/`, forming long-term memory that survives compaction and spans sessions. s10 splits the system prompt into sections and assembles them at runtime according to the current context, avoiding one hardcoded, unmaintainable chunk of prompt.

A shorter version:

> The theme of s06-s10 is: protect the context, load knowledge on demand, preserve important information long term, and assemble the system prompt dynamically.

---

## 11. Self-Test Questions

### Question 1

After the main Agent calls `task`, the sub-agent internally reads 10 files and runs bash 5 times. Will the main Agent's `messages` keep the details of those 15 steps?

Answer: no. The main Agent only receives the final summary returned by `spawn_subagent()`. But the sub-agent's real side effects, such as writing files, are kept.

### Question 2

Can the model see `SKILL_REGISTRY` directly?

Answer: no. It is a dictionary in Python memory. It must be converted to plain text by `list_skills()` and then sent to the model via `system=SYSTEM`.

### Question 3

Why doesn't s07 stuff all the `SKILL.md` files into `SYSTEM`?

Answer: because most tasks do not need all the skills, and the full text would waste tokens and create noise. s07 puts only the catalog in and loads the full text on demand with `load_skill`.

### Question 4

Why does s08 write large tool_results to disk?

Answer: large outputs would blow up the context. Writing to disk preserves the complete content while `messages` keeps only the path and a preview.

### Question 5

Do `compact_history()` and `extract_memories()` have the same goal?

Answer: no. `compact_history()` exists to shorten the current context; `extract_memories()` exists to preserve important preferences, constraints, and project facts long term.

### Question 6

Why does s09 use `pre_compress` for memory extraction?

Answer: because the compacted `messages` may already have lost the details. `pre_compress` is a pre-compaction snapshot that lets memory extraction see more complete information.

### Question 7

When `.memory/MEMORY.md` does not exist, will s10 add the memory section to the system prompt?

Answer: no. `update_context()` yields `memories = ""`, so the `if memories` check in `assemble_system_prompt()` fails.

### Question 8

Is s10's cache the same thing as an API prompt cache?

Answer: no. The teaching version's `get_system_prompt()` only avoids repeatedly concatenating strings locally; an API prompt cache is a server-side mechanism for caching prompt tokens.

---

## 12. Code Location Index

### s06

- `s06_subagent/code.py:58` - `SUB_SYSTEM`
- `s06_subagent/code.py:182` - `SUB_TOOLS`
- `s06_subagent/code.py:196` - `SUB_HANDLERS`
- `s06_subagent/code.py:201` - `extract_text()`
- `s06_subagent/code.py:207` - `spawn_subagent()`
- `s06_subagent/code.py:210` - the sub-agent's fresh `messages`
- `s06_subagent/code.py:212` - the sub-agent's 30-round cap
- `s06_subagent/code.py:249` - returning only the summary
- `s06_subagent/code.py:252` - `task` tool schema
- `s06_subagent/code.py:257` - `task` handler mapping

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
- `s07_skill_loading/code.py:304` - `load_skill` handler mapping

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
- `s08_context_compact/code.py:454` - the compaction insertion point in `agent_loop()`
- `s08_context_compact/code.py:488` - the model calling compact on its own

### s09

- `s09_memory/code.py:42` - the `.memory/` path
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
- `s10_system_prompt/code.py:195` - recomputing context/prompt after a tool round
- `s10_system_prompt/code.py:204` - initial `context = update_context({}, [])`

---

## 13. Recommended Review Order

First review pass:

```text
1. Read section 1, the overall storyline
2. Read section 7, the side-by-side comparison
3. Read s06 and s07
4. Read s08 and s09
5. Read s10
6. Do the self-test questions in section 11
```

Second review pass:

```text
1. Look only at the code index in section 12
2. Jump to the corresponding source lines
3. Say out loud what problem each function solves in the whole Agent harness
```

Third review pass:

```text
Draw this line yourself:

complex task -> task -> subagent -> summary
domain knowledge -> skill catalog -> load_skill
context too long -> compact pipeline
important information -> extract_memories -> .memory
system prompt -> context -> prompt sections -> SYSTEM
```

If you can draw it, you have mastered the skeleton of `s06-s10`.