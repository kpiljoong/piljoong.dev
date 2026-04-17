+++
date = '2026-04-18T01:15:00+09:00'
draft = false
title = "From Prompting to Context. Still Not the System."
description = "AI made it easy to produce code, but not to keep systems in order. Prompting and context are not enough. What’s missing is a layer that can hold state and turn claims into accepted truth."
tags = ["ai", "engineering", "software-design", "systems", "architecture", "agents", "state"]
categories = ["engineering", "architecture"]
slug = "from-prompting-to-context-still-not-the-system"
+++

Early on, prompting felt like the core problem.

If you wrote better prompts, you got better results. That framing made sense for a while.

Now the conversation has moved toward context engineering instead: better retrieval, better grounding, better input shaping. I think that shift is real. It is also incomplete.

AI made it much easier to produce code.

But it did not make software work less stateful.

That is the part I keep coming back to. The more I use coding agents, the less I think the interesting problem is the prompt itself, or even the context alone. A good prompt can produce a good patch. Better context can produce a better one. But the work around that patch still has shape: intent, constraints, assumptions, decisions, evidence, review, acceptance.

If those things only live in the chat, they disappear too easily.

And if they disappear, the system can keep changing while our understanding of it does not. What is missing is not better prompting or better context alone. It is the ability to hold state.

## Prompting Is Too Small a Unit

A prompt is a useful instruction.

It is not an operating model.

That distinction matters more as agents become more capable. When an agent only writes a small function, the surrounding process can stay informal. You read the diff, run the test, maybe ask for a small follow-up.

But once an agent is changing a real system, a single prompt is too small to hold the work.

What is the goal?
What do we know about the current system?
Which constraints should not be violated?
What has been accepted as true?

Those questions do not fit cleanly into a prompt.

More importantly, they should not be trapped inside one. Context helps shape the input. It does not, by itself, create durable state.

## Chat History Is a Weak Form of State

Chat feels persistent because the messages are still there.

But that does not make it a good state model.

I have had long agent sessions where the useful context was technically somewhere in the conversation, but practically gone. A constraint was mentioned forty turns ago. A design direction changed in the middle. The agent produced a plausible explanation, then a patch, then another patch. Each step made sense locally.

After enough steps, the work was no longer organized around a clear model.

It was organized around accumulated conversation.

That is fragile. Not because chat is bad, but because chat is too loose. It mixes intent, evidence, speculation, commands, intermediate output, and final decisions into the same stream.

When everything is a message, nothing has a role.

## Agents Produce Claims. Systems Need Truth.

One of the most useful distinctions I arrived at was this:

agent output is a claim.

It may be useful. It may be correct. It may be based on real repo exploration. But until something checks it, records it, promotes it, or rejects it, it is still a claim.

This matters because coding agents are very good at producing coherent output.

They can analyze a repository and produce a convincing summary. They can propose a plan. They can write a build spec. They can review their own patch. All of that can be useful.

But a coherent artifact is not automatically truth.

For real work, I want the system to know the difference between raw analysis, validated evidence, a proposed plan, an approved decision, a build instruction, a review result, and an accepted outcome.

Those are different states. They need different authority.

## One Prompt at a Time Becomes Patch Accumulation

The failure mode is usually not dramatic.

It starts with a reasonable request.

Then a correction.

Then a constraint that should have been explicit from the beginning.

Then a test failure, followed by a small refactor to make the test pass.

Then a follow-up because the previous patch changed an adjacent behavior.

None of these steps are obviously wrong. In fact, this is exactly why prompt-driven work feels so productive. The loop is fast. The system responds. Code changes appear quickly.

But after enough iterations, the work can start to drift.

I have seen systems drift this way without anyone explicitly choosing that outcome.

The final code may no longer correspond to a design anyone intentionally chose. It corresponds to the path taken through the conversation.

At that point, you are no longer evolving a system through a clear model. You are replaying a conversation.

## The Worker Should Not Own the Gate

I do not want the same opaque agent output to own the whole lifecycle.

An agent can explore. It can draft. It can propose. It can implement. It can review.

But if the workflow collapses into “ask the agent and accept the answer,” the control boundary is gone.

This is especially important when reusing one agent across multiple stages. Keeping one worker context warm can reduce handoff pain. The agent remembers more. It reads the repo less repeatedly. It can carry context from analyze to plan to build to review.

That is useful. But the workflow should not collapse into one long opaque response.

Each stage still needs a boundary. Each request needs a shape. Each response needs a place to land. Each output needs to be interpreted with the right level of authority.

The agent can produce a claim.

The system has to decide what that claim becomes.

## Stages Are Not Ceremony

I used to worry that adding stages would make the agent workflow feel heavy.

Sometimes it does.

For small tasks, ceremony is worse than the problem. If I am generating a small viewer, a throwaway script, or a narrow internal utility, I do not need a full architectural workflow around it.

But for changes that need to survive, stages are not bureaucracy. They are how the work stays understandable.

Analyze is not plan.

Plan is not build.

Build is not review.

Review is not acceptance.

Those boundaries sound obvious when written down. They become easy to lose when an agent can jump from repository exploration to confident implementation in the same interaction. Speed makes boundaries easier to skip, which is exactly why I want the boundaries to be explicit.

## Artifacts Matter Because Memory Is Not Enough

I do not want an agent workflow where the only durable artifact is the final diff.

The diff tells me what changed. It does not tell me enough about why the change was chosen, what constraints shaped it, what evidence supported it, which risks remain, or what was accepted as true.

That is why I want artifacts.

Not artifacts as paperwork. Artifacts as durable state.

A session can hold intent. A decision can hold commitment. A plan can describe the selected path. A build spec can guide implementation. A review or lock report can freeze what was checked.

The exact filenames matter less than the role. The point is that work should leave behind structured memory.

## What I Started Calling Ordo

That gap is not about better prompts or better context.

It is about what turns a claim into something the system can treat as truth.

That missing layer is what I started calling Ordo.

At first, it looked like a CLI.

But that was never the core idea.

The core idea was order: keeping AI-assisted work in a known, controlled state.

Structuring the work around stages, artifacts, and decisions, instead of treating it as a sequence of prompts.

In that model, Ordo owns meaning. The runtime can execute. An orchestrator can call agents. A chat surface can control sessions remotely. But the semantic layer has a different responsibility: it defines what the work is, what state it is in, and what must be true before the work can continue.

The separation between semantic meaning and execution matters.

If execution owns meaning, the system becomes whatever happened at runtime.

## Ordo Does Not Replace the Agent

The point is not to make the agent less capable.

The point is to make the work more operable.

Ordo does not need to replace a repo-reading agent. In fact, it can use one. A strong external agent is useful for exploration, planning, building, and review.

Ordo does not replace the agent.

It restores that boundary.

But Ordo should preserve contracts, assemble context, record outputs, enforce gates, and keep claims separate from accepted truth.

That is a different job. The agent does the work. The system keeps the work in order.

## Better Prompts Won't Fix Missing State

I still care about prompts.

Clear instructions matter. Good context matters. A better prompt can produce a much better result.

But better prompts do not solve the state problem by themselves.

Better context does not solve it either.

They do not automatically give you durable intent. They do not preserve decision history. They do not separate claims from truth. They do not define acceptance. They do not create a reliable boundary between stage output and final outcome.

For that, you need a system around the agent.

This is the shift I am trying to make with Ordo.

Not from bad prompts to better prompts.

From prompting to controlled work.

From chat history to durable state.

From agent output to staged artifacts, explicit decisions, and checkable transitions.

The goal is not to slow the agent down.

The goal is to keep the work in order while it moves fast.

Better prompts were never enough.
Better context is not enough either.

The missing layer is the system that keeps work in a known, controlled state while it changes.

That is the layer this kind of work will need as it becomes normal.

That is what I started calling Ordo.
