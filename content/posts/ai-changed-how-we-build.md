+++
date = '2026-04-07T21:07:53+09:00'
draft = false
title = "AI Changed How We Build. It Did Not Change What Matters."
description = "AI made it cheap to build software. It did not make it safe to run."
tags = ["ai", "engineering", "software-design", "operations", "governance", "systems"]
categories = ["engineering", "architecture"]
slug = "ai-changed-how-we-build"
+++

*A view on engineering in the AI age*

AI is powerful. That is no longer a theoretical point.

I spend far less time writing code from scratch than I used to. With a short prompt, I can generate something that would have taken hours or days before. I use that constantly now, and I don't think there is any serious way to deny that the economics of building software have changed.

But after using it on real systems, one thing became obvious to me.

AI changed how we build.  
It did not change what makes systems work.

## Building Got Cheap

For a long time, implementation was the expensive part. Writing code, connecting parts, putting together internal tools, building small utilities people needed but never quite had time for. A lot of that work is simply cheaper now.

If I need a small dashboard, a viewer, or some narrow internal tool, I can often get something usable in a few hours. In that kind of work, I do not always read every line anymore. Sometimes it is faster to fix it after the fact, or even regenerate it, than to inspect it up front.

That is a real shift. It matters.

But I have also learned that this only applies to a certain category of software. Once the system has to survive real use, the standard changes.

## I Still Don't Trust It

For anything that looks like a real service, I still want to understand what the system is doing, where it can fail, and how it behaves when conditions stop being ideal.

If I cannot explain that clearly, I do not think of it as production-ready.

That is not because I am skeptical of AI in some abstract sense. It is because I have spent enough time around systems that looked fine until they were under load, partially degraded, or just slightly outside the conditions people had in mind when they built them.

That kind of failure did not go away.

I ran into this recently in a launch review. The system did not fail. That was the problem. It returned an empty result instead, which made the behavior look safe from the outside. Nothing crashed. Nothing alerted. But the fallback was really just a way of hiding failure. The system had moved into a state where it could be wrong without making that visible. Once failure gets turned into something a system can silently absorb, I stop trusting the code no matter how clean it looks.

If anything, AI makes it easier to produce something that looks complete before it has really earned that confidence. Generated code is easy. Owning a system is not.

## Someone Still Owns the Failure

There is a common narrative that AI reduces the need for engineers. I do not think that is what is happening.

What AI reduces is the friction of building. That is different.

When it becomes cheap to produce software, more software gets produced. More decisions get embedded in what the model handed back. More things start to look "done" earlier than they should. And someone still has to decide whether the thing is safe to run, safe to depend on, and safe to evolve.

That responsibility does not disappear. In practice, I think it increases.

The easier it is to build, the easier it is to create systems whose risks nobody has really examined.

## SaaS Is Not Dead, but the Boundary Is Moving

I do not think SaaS disappears. But I do think the balance shifts.

We will build more things ourselves. Not because every company suddenly wants to become a software vendor, but because it becomes much easier to create tools that match a particular team, workflow, or context. Things that used to be too small to justify building are now cheap enough to make locally.

I already see that happening.

But when that boundary moves, responsibility moves with it. If I generate a tool myself instead of buying one from a vendor, then I own the behavior. I own the failure modes. I own the consequences when it breaks at the wrong time, or silently does the wrong thing.

That is not a minor shift. It is a change in who is accountable.

## AI Raises the Baseline. It Does Not Raise the Standard.

One thing I have noticed is that AI-generated systems often look better than a lot of software that teams were already shipping.

You often get the obvious patterns for free. Basic retries. Some error handling. Cleaner structure than whatever rushed internal tool would have been written otherwise. In teams that were weak on engineering discipline to begin with, this can genuinely improve the baseline.

But that is exactly why it can be misleading.

A better-looking baseline is not the same thing as what I would actually trust in production. I have seen systems that had all the recognizable patterns and still failed badly under real conditions. Not because the code looked messy, but because the assumptions were wrong, the boundaries were unclear, or nobody had really thought through what would happen once the system was exposed to change and failure.

Production systems are not defined by whether they contain familiar patterns. They are defined by how they behave when things stop going well.

That still takes deliberate design.

## The Hard Part Is Control

The biggest change I see in the AI era is not productivity. It is control over what gets shipped.

AI makes it easy to produce software quickly. Much easier than before. What it does not make easy is deciding whether that software should exist, whether it is correct enough to rely on, and whether the organization can still reason about it once it starts changing.

That is the part that worries me more.

Without control, bad assumptions spread quickly. Inconsistencies accumulate. Failure modes become harder to trace because too much was generated too quickly and not enough of it was made explicit. I have already seen cases where something looked fine, worked for a while, and then became difficult to reason about precisely because no one had a strong model of why it worked in the first place.

## Speed Only Helps If Validation Keeps Up

Faster iteration is valuable, but only when our ability to check it keeps up.

If the cost of building drops, then our ability to verify, observe, and challenge what gets built has to improve as well. Otherwise, all you have done is increase the rate at which unexamined systems enter the world.

For me, that is the practical question behind AI-assisted engineering. Not "can we generate this?" but "how do we keep a fast build loop connected to reality?"

That usually means explicit checks, observable behavior, and feedback loops tied to real outcomes rather than appearances. It means being able to tell whether the system is actually correct, actually safe, and actually aligned with how it will be operated.

I think this is where a lot of teams will struggle. The bottleneck has moved, but many people are still looking for it in the old place.

## The Old Practices Still Matter, but They Do Not Transfer Cleanly

Reliability practices still matter. Architecture still matters. Operational discipline still matters.

None of that became irrelevant.

But the context changed. Systems are created faster. Assumptions are now embedded in prompts and in what the model produced rather than discussed explicitly in design docs or code review. Change becomes more continuous, and often less visible.

So I do not think the right response is to throw away existing engineering practice. I think the right response is to adapt it.

A new model of building requires a new operating model around validation, control, and change.

## Engineering Did Not Disappear

People sometimes say developers will no longer need to code. In a narrow sense, I understand what they mean. I do spend less time typing code than I used to.

But that does not make engineering less important. If anything, it makes engineering more exposed.

I spend less time on syntax and more time thinking about system behavior, failure modes, trade-offs, and boundaries. And those questions do not get easier in the kinds of systems that matter most. Concurrency is still hard. Backpressure is still hard. Connection management is still hard. Distributed failure is still hard.

AI can apply patterns. It cannot decide when those patterns matter, what trade-offs they imply, or what happens when reality pushes back.

That still requires understanding.

## This Is Where Prompt-Driven Systems Start to Drift

Another thing AI does not change is that systems do not stay still.

Requirements move. Constraints move. Teams move. The shape of the problem changes while the system is still alive. That is where I have seen prompt-driven development become fragile. Context gets compressed. Assumptions pile up. Prompts drift. Side effects show up later, when nobody remembers exactly why the system ended up this way.

I have seen this happen in a very ordinary way. A system starts simple, then gets patched one prompt at a time: a timeout here, a new state there, background polling, retry, recovery, cancellation. Each change is locally reasonable. But after enough iterations, the shape of the system starts to drift. What began as a tool quietly turns into something closer to a platform, except nobody explicitly designed it that way. The system keeps growing, but the model of the system does not keep up.

For small tools, this is often acceptable.

For systems that need to survive change, I have found that design still has to come first. Not design in the sense of heavyweight process, but in the sense of having a model of what the system is, where its boundaries are, and what kinds of failure it needs to withstand.

Then AI can help implement that design much faster.

## A Different Direction

If building becomes cheap, then it becomes more reasonable to ask whether our tools actually fit the problem.

For a long time, adopting a new language or runtime carried a heavy implementation cost. Even when something matched the problem better, the surrounding cost was often too high. AI changes that equation, at least somewhat. When more of the implementation work can be generated, the question shifts from "can we afford to build this?" to "does this model fit the problem better?"

That is part of what led me to work on BUSAN: not because novelty matters, but because cheaper implementation opens up design space that used to be too expensive to explore.

## So What Actually Changed?

AI did not remove the need for engineering.

It changed where engineering matters.

Building became easier. Designing did not. Operating got harder. Governing what gets built became critical.

That is the shift I see.

We are less constrained by how fast we can produce software. We are more constrained by how well we understand it, validate it, and keep it under control once it starts changing.

This is not a tooling problem. It is an operating problem.

## What Comes Next

This is the frame I am using for the AI era.

In the next posts, I want to go deeper into what that means in practice: how to operate coding agents as systems rather than chat tools, how to think about control and risk when change becomes cheap, how to build feedback loops that keep fast iteration safe, and how to design runtimes and systems that match the real shape of the problem.
