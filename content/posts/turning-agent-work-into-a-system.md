+++
date = '2026-05-11T02:55:00+09:00'
draft = false
title = "Turning Agent Work Into a System"
description = "AI agents can produce useful output, but useful output is not system state. Ordo is a model for durable artifacts, explicit decisions, gates, and accepted truth."
tags = ["ai", "engineering", "software-design", "systems", "architecture", "agents", "state"]
categories = ["engineering", "architecture"]
slug = "turning-agent-work-into-a-system"
images = ["/images/og-turning-agent-work-system.png"]
+++

In the last post, I argued that prompting is not the system.

The missing layer was the system that keeps work in a known, controlled state while it changes. That is what I started calling Ordo.

This post is about the shape that idea took.

Better prompts help. Better context helps more. But neither one gives the work durable state, explicit authority, or clear transitions. That was the gap I kept running into, and Ordo was my attempt to give the work a shape it could keep.

## I Did Not Start With a Platform

I did not start by trying to build a platform for AI-assisted development. I started with a smaller discomfort: I wanted the work to stay understandable after the agent had already done something useful.

The problem became obvious after a few successful sessions. An agent would read a repository, propose a plan, write a patch, explain what it changed, and then respond to a follow-up. Each step made sense. The output was useful. The patch often worked.

But when I came back later, the state of the work was harder to name than it should have been. I could see what the agent had said, but I could not always tell which parts were repo evidence, which parts were proposed direction, which parts had become decisions, and which parts were just plausible reasoning from that moment.

That was the part I wanted to fix. I was not trying to make the agent smarter. I was trying to stop the work from dissolving into conversation.

## The First Visible Shape Was a CLI

The early version of Ordo looked like a CLI. It created repository-local docs, opened bounded sessions, recorded decisions, checked expected context, and generated prompts with compact context views.

That was useful, but it was not the core idea. The CLI was only the first visible shape of the model.

What I actually wanted was stricter than a tool. I wanted a way to keep AI-assisted work in a known, controlled state while the actual worker execution happened somewhere else. A tool helps you do the work. A system defines what the work is, what state it is in, and what is allowed to happen next.

## The Repository Had to Become State

The first durable thing I trusted was not the chat history. It was the repository. If intent, decisions, evidence, and boundaries mattered, they needed a place to live that was durable, inspectable, and versioned with the work.

Not in memory. Not in an opaque agent session. Not in whatever the model happened to say last.

That is how the core artifact roles became clearer. A `session` holds bounded work and the current summary of that work. A `decision` records an explicit commitment. `domain` keeps durable operating context, boundaries, and contracts. `canonical` represents curated repo-wide truth after explicit promotion.

Stage artifacts then grew around those roles: analysis records, plan and build specs, review and acceptance records, critique and verification outputs, finalization reports, and other evidence produced along the way.

The filenames are not the point. The point is that the work leaves behind durable state with roles.

That is what chat history cannot do well. Chat preserves text. It does not preserve authority.

## The Work Needed Roles, Not Just Messages

One of the easiest mistakes in agent workflows is treating all useful output as if it has the same meaning. It does not.

A repository summary, a decision, a plan, a build instruction, and a review result are not just different documents. They carry different authority. A summary can describe what the agent saw. A decision can commit to a direction. A plan can propose a path. A build spec can tell the worker what to change. A review or critique can say what was checked and what still looks risky.

This is where stages stopped looking like ceremony and started looking like structure. I did not add stages because I wanted a process diagram. I added them because work changes meaning over time. If the system does not distinguish exploration from planning, planning from implementation, and implementation from review or acceptance, then it cannot tell the difference between a useful claim and accepted truth.

That is when velocity starts to produce ambiguity instead of progress.

## Agent Output Is Still a Claim

The sentence that clarified the model for me was simple: agent output is still a claim.

That was the point of the previous post, so I will not repeat the whole argument here. The important part for Ordo was what follows from it. If agent output starts as a claim, then the system needs a way to decide what happens to that claim.

Some claims stay as evidence. Some become proposed decisions. Some are rejected. Some become accepted within a bounded project. A smaller set may later be promoted into repo-wide canonical truth.

That last distinction matters. Not every accepted result should automatically become durable repo truth. Finalizing a piece of work means the bounded workflow is complete. Promoting knowledge is a separate boundary.

```text
agent output
  -> staged artifact
  -> gate
  -> bounded acceptance
  -> optional explicit promotion
  -> repo-wide current truth
```

That is the difference between moving quickly and letting every convincing answer become part of the system's memory.

## The Model Needed Gates

Once I started thinking in terms of claims and truth, gates followed naturally.

A worker can produce an artifact. That does not mean the work should continue. The system still needs to know what is unresolved, what blocks the next stage, what requires review, and what is safe to accept.

That is what a gate is doing here. It is not there to make the process feel heavy. It is there because continuation has meaning.

That was the standard I wanted.

## Ordo Owns Meaning

This is the sentence that ended up clarifying the architecture: Ordo owns meaning.

By that I mean Ordo defines the contract of the work. It decides what artifacts exist, which ones carry authority, when a stage is valid, when work can continue, what finalization means, and what is allowed to become current truth.

Execution is a different concern. A workflow layer can invoke tools. An orchestrator can manage retries, approvals, and resume logic. A chat surface can control sessions remotely. Those things matter, but they should not be allowed to redefine what the work means.

The reason is simple: if the runner defines the meaning of the work, every runtime becomes its own truth model. The system becomes whatever happened to execute last.

I did not want that. I wanted the semantics to survive execution.

## Why Ordo and ordo-flow Are Separate

This boundary eventually forced a split.

Ordo should not be the thing that runs every command. And the runner should not be the thing that decides what the work means.

Once that became clear, the split stopped looking optional. Ordo is the semantic layer. `ordo-flow` is the workflow and orchestration layer.

Ordo defines artifact roles, semantic state, validation, gates, finalization, and the boundary between bounded acceptance and promoted repo truth. `ordo-flow` moves work through the staged workflow, calls tools and models, manages retries and resume, surfaces approvals, and maintains the active execution path.

One owns meaning. The other owns workflow execution.

That split also makes stale work easier to reason about. Ordo can say that downstream artifacts from an older path should stop counting as current truth after an earlier stage is rerun. The workflow layer can then do the operational work of clearing, replacing, or withdrawing those artifacts so the active path stays coherent.

That is a better separation than asking one workflow layer to invent both the policy and the execution behavior at once.

## I Still Wanted Warm Context

There was still a practical tension to solve. A single strong agent is often useful across many stages. It can analyze, plan, build, and review with less handoff pain than a fragmented pipeline.

I did not want to give that up. I also did not want one warm agent session to become the workflow itself.

That is why the stage boundary still mattered even when the same worker was reused. The same worker can move through multiple stages, but each stage still needs a request shape, an expected output shape, a place for the result to land, and an authority boundary around that result.

That keeps the context warm without letting the workflow dissolve into one opaque thread.

The same issue shows up with compact context. I want later work to start from a compact view when that is enough. I do not want the compact view to become the new source of truth.

The useful shape is compact context with linked evidence. Start from what has been accepted, keep the source refs, and follow the linked artifacts when the system needs to understand why something became true.

## The Goal Was Never More Automation

At this point, I do not think of Ordo as a way to automate prompting. The commands, docs, workflows, and integrations all sit downstream of the real job: keeping the work inspectable while it moves.

That means preserving durable intent, explicit decisions, bounded authority, structured evidence, checkable transitions, and current truth that is separate from drafts and claims.

Those are not productivity features in the usual sense. They are the properties that keep fast agent work from becoming a pile of plausible output.

## Turning Agent Work Into a System

The more I worked with coding agents, the less I believed that the hard part was asking better questions. The hard part was giving the work a shape it could keep.

That is what Ordo became for me. The name was intentional. Ordo comes from the idea of order: not making the agent smarter, but keeping the work arranged, bounded, and inspectable.

So I do not think of it as a better prompt wrapper or a smarter chat interface. I think of it as a way to turn agent work into a system with durable state, explicit authority, and controlled transitions.

I have been shaping that model in the Ordo reference repository:

https://github.com/kpiljoong/ordo

The agent can move fast.

The work still needs a system that remembers what became true.
