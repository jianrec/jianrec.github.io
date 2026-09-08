---
title: "Core Skills Every Developer Needs in the AI Era"
date: 2026-06-28 13:20:00 +0800
tags: [AI Coding, AI-Native Developer, Software Engineering, Agent, T-Shaped Developer, DORA, Engineering Productivity]
main_category: "Paper Reading"
sub_category: "Agent · AI Coding"
discipline: "Agent"
course: "AI Coding"
material_type: "Video Notes"
description: "Notes on the Google for Developers talk Build core skills to thrive as an AI-era developer: the core capability of an AI-era developer is not writing code faster, but clarifying intent, designing systems, validating output, and organizing feedback."
lang: en
---

> Original PDF: [Open the PDF]({{ '/assets/pdfs/ai-era-developer-core-skills.pdf' | relative_url }})

These notes are drawn from the Google for Developers talk **Build core skills to thrive as an AI-era developer**. The speakers are Nicole Forsgren and Andrew MacVean, and the video was published on 2026-05-21.

One essential question: now that AI can dramatically speed up code generation, what capability is genuinely scarce in a developer?

My reading is: clarifying intent, designing the environment, establishing feedback, holding on to judgment, and enabling the team to absorb a higher density of change. Why are we doing this? Who is it for? How should it be done? What are the boundary rules?

## 1. Once AI Speeds Things Up, the Bottleneck Is No Longer Just Writing Code

The observation: AI adoption inside engineering teams is already high, but high usage does not automatically translate into higher overall team productivity.

The reason is simple. Software engineering has never been only about writing code. An idea passes through requirements, design, implementation, validation, release, and feedback, with many steps in between. Once AI makes local coding faster, the problems that used to be masked by slow coding become far more visible:

- Whether the requirements are clear enough.
- Whether system boundaries are well defined.
- Whether context can be shared across the team.
- Whether quality assurance can keep up.
- Whether feedback returns to engineering decisions in time.

If these steps are not redesigned, AI generating more code will only spread communication cost, rework cost, and quality risk more quickly.

So what AI brings is not simply a "coding speed problem," but a question of "whether the engineering system can absorb faster execution."

## 2. Five Patterns of High-Performing AI-Native Engineers

The talk summarizes the behavior of high-performing AI-native engineers into several patterns. Compressed into a single thread, it is a shift from writing code locally to running an end-to-end system.

First, stand higher up.

Developers should care not only about how, but also about why, for whom, and what the criteria for success are. The more capable AI is at executing, the earlier human judgment must enter the problem-definition stage.

Second, clarify intent earlier.

The most important concept in the talk is **shift left on intent**. Traditionally, shift left is applied to testing, security, and quality; in AI-native development, what needs to shift left is goals, constraints, trade-offs, and success criteria.

Third, design the environment rather than just demanding code.

If an agent has no context, no rules, no tests, and no rollback path, then no matter how fast it generates, it will struggle to produce high-quality results consistently. The engineering environment itself becomes part of productivity.

Fourth, shorten the execution and feedback loop.

AI lets small teams handle more, but only if the team has a short enough feedback loop. User feedback, validation feedback, performance feedback, and retrospective findings should all be systematically written back into the development process.

Fifth, keep learning with an experimental mindset.

AI workflows will fail. The point is not to avoid all failure, but to make failure observable, reviewable, and convertible into team knowledge.

## 3. "Shift Left on Intent" Is the Most Important Concept in the Whole Talk

As more and more execution is handed to AI, code is no longer the single source of truth. Upstream specifications, constraints, business context, risk boundaries, and success metrics become the key inputs that determine whether AI does the right thing.

If intent is not expressed explicitly, AI will amplify vague requirements into output that is faster but messier. The tacit knowledge that used to live in the heads of senior engineers needs to be gradually converted into explicit knowledge that both the team and the agents can share.

A specification written for AI collaboration should at minimum make the following clear:

```yaml
goal: Reduce latency for the checkout confirmation page
target_users: Mobile users in unstable network conditions
constraints:
  - Do not regress payment correctness
  - Preserve observability and rollback hooks
success_criteria:
  - p95 latency reduced by 30%
  - no increase in failed confirmations
review_checks:
  - edge cases documented
  - red-team scenario evaluated
```

This kind of specification is not about adding documentation overhead. It is about keeping "judgment" in human hands so that "execution" can be handed to agents with more confidence.

## 4. The T-Shaped Developer in the AI Era: Depth Still Matters, Breadth Is Amplified

The vertical stem is still depth in software engineering: architecture, reliability, understanding complex systems, quality judgment, and engineering trade-offs. AI has not reduced the importance of these skills. Quite the opposite: the more AI participates in execution at scale, the more human engineers need deep technical intuition to judge whether the system is racing in the wrong direction.

The horizontal bar expands noticeably, mainly across three parts:

- AI collaboration skills: knowing what context to give AI, knowing which tasks it is unreliable at, knowing how to validate its output, and knowing when a human must make the final call.
- Adjacent engineering skills: understanding deployment, observability, experiment design, architectural evolution, risk control, and engineering guardrails.
- Adjacent non-engineering skills: understanding users, business goals, organizational constraints, and value judgments.

This does not require developers to master everything. It requires them to keep their professional depth while connecting AI, the engineering system, and business goals into a closed loop.

## 5. Delegate Tasks, Not Judgment

An AI-native development principle:

> delegate tasks, not judgment

In other words, you can delegate tasks to agents, but do not outsource your judgment.

AI can take on many tasks that are describable, verifiable, and constrainable: generating candidate implementations, filling in tests, organizing logs, migrating code, analyzing anomalies, drafting initial documentation. But the following judgments must still be owned by humans:

- Whether the success criteria are defined correctly.
- Whether the current output satisfies real-world scenarios.
- Whether risk and reward are proportional.
- Whether the system is drifting from the original intent.
- Whether the quality bar should be raised or lowered.

What truly matters is task decomposition, context design, boundary control, and result validation.

## 6. From a Single Agent to a Balanced Agent Team

The second half of the talk discusses multi-agent systems, but with a very pragmatic stance: do not blindly chase agent count; chase clear role boundaries, stable scopes of responsibility, and well-defined output interfaces.

Multi-agent setups suit complex task decomposition, but they also create new problems:

- Overlapping agent responsibilities.
- Confused context sources.
- Conflicting outputs.
- Difficulty locating responsibility for failures.
- Teams assuming "more automation" equals "a better system."

So the better direction is not agent sprawl, but building a balanced agent team: each agent with a clear task, clear inputs, clear outputs, and a clear way to be validated.

## 7. Guardrails, Red Teaming, and Observability Must Level Up Too

If AI raises delivery speed by many multiples, then system diagnosis and guardrail capabilities must rise in step. Otherwise the team is just producing harder-to-debug problems faster.

The talk emphasizes several very engineering-oriented practices:

- Use blameless retrospectives to train the team's intuition for edge cases and failure modes.
- Use red-team agents to proactively find holes, rather than waiting for problems to surface in production.
- Use whiteboards and manual walkthroughs to build an architectural mental model before writing code.
- Use observability tools and data agents to help engineers understand complex systems.

AI can be used not only to generate systems, but also to understand them.

As logs, traces, performance data, and exception stacks become increasingly complex, AI should help engineers diagnose, attribute, and build system understanding. Otherwise faster development speed is traded for higher debugging cost.

## 8. Developers Should Become Value Translators

As AI handles more and more of the "how," developers need to focus more on the "what" and the "why."

For example, a stakeholder says "improve performance," and AI can immediately optimize an inefficient loop for you. But the questions that really matter may be:

- Who is perceiving the performance problem?
- On which devices, networks, and paths is it worst?
- Which metric best represents user experience?
- How does this optimization rank in priority against other requirements?
- What may be sacrificed, and what may not?

This is a developer's value translation skill: turning vague business goals into engineering goals that are executable, verifiable, and constrainable.

The talk also warns against dumping all user feedback into an LLM and only reading the summary. Summaries are valuable, but users' tone, hesitation, confusion, and anger also carry first-hand signals needed for product judgment. If engineers lose all contact with real user pain points, their sense of "why this is worth doing" weakens.

## 9. Managers Cannot Just Demand That Individuals Level Up

The talk closes by pulling back to the organization: you cannot force everyone to become a T-shaped developer inside a broken system.

If a team already has fragmented tooling, knowledge silos, a blame culture, and misaligned metrics, AI will not fix those problems automatically. It will only expose them faster.

Managers need to do at least three things:

- Redefine productivity metrics: do not just look at PR counts, throughput, or lines of code; look at outcomes, quality, and business value.
- Protect "productive struggle": give the team time to understand tools, walk through architecture, and build mental models, instead of only pushing for delivery.
- Build psychological safety: allow agent workflows to fail, and convert failure into collective learning through blameless retrospectives.

Team management in the AI era is essentially managing cognitive load. Engineers are not just writing code; they are validating machine output, making architectural judgments, and switching context across multiple agents and tools. If the organization does not reduce wasteful switching, does not consolidate key learning, and does not let feedback flow back, the team easily moves from high efficiency into high pressure.

## 10. Summary

The most valuable thing about this talk is that it does not frame the change facing AI-era developers as a tooling upgrade, but as a restructuring of the engineering role.

AI makes execution cheaper, so the question is no longer only "can it be done," but "what to do, why to do it, and how to do it reliably."

Because execution is cheaper, intent, context, rules, and feedback become more expensive.

Because speed is higher, system boundaries, guardrails, observability, and organizational culture must be stronger.

Because AI has taken over more of the how, developers must take on the what and the why more deeply.

The people who will keep producing high value in the AI era are not necessarily those who generate code the fastest, but those who are best at designing systems, clarifying intent, holding on to judgment, and driving organizational learning.

## Sources

- Video: Build core skills to thrive as an AI-era developer
- Channel: Google for Developers
- Speakers: Nicole Forsgren / Andrew MacVean
- Published: 2026-05-21
- Duration: 44:18
- Video link: [https://www.youtube.com/watch?v=q_Jq4IgYImk](https://www.youtube.com/watch?v=q_Jq4IgYImk)
- Original notes PDF: 成为 AI 时代开发者必备的核心技能.pdf