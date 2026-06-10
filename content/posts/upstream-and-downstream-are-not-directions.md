+++
date = '2026-06-11T03:04:06+09:00'
draft = false
title = "Upstream and Downstream Are Not Directions"
description = "Upstream and downstream sound precise, but they only work after teams agree which flow they mean."
tags = ["software-engineering", "architecture", "systems", "incident-review", "communication", "distributed-systems"]
categories = ["engineering", "architecture"]
slug = "upstream-and-downstream-are-not-directions"
images = ["/images/og-upstream-downstream-are-not-directions.png"]
+++

The incident review started with a sentence that sounded ordinary.

> "The downstream service returned bad data."

Everyone knew what the speaker meant.

For about thirty seconds.

Then the conversation started to split.

The display team was talking about the pricing service they called during page rendering. From their point of view, pricing was downstream. The request left the display path, crossed a service boundary, and came back with price data.

The pricing team heard something different. They thought of themselves as upstream because they produced the data the display layer consumed. The price came from their system. The display page was downstream of that data.

The logistics team had a third interpretation. They were looking at the end-to-end commerce flow: discovery, display, checkout, payment, fulfillment, settlement. In that map, display was upstream of checkout, checkout was upstream of fulfillment, and fulfillment was downstream of almost everything before it.

Nobody was being careless.

The word was doing too much work.

That should not have surprised me as much as it did.

Software engineering runs on metaphors. We talk about layers, pipelines, streams, queues, ownership, contracts, boundaries, and gateways because the real thing is invisible. A program has no physical layer. A service does not literally own a field. A queue is not always a line of people waiting at a counter.

The metaphors work because they compress a mental model. They fail when that model is missing.

Extreme Programming had a practice called "system metaphor": a simple shared story that helped customers, programmers, and managers talk about the system in the same language. The important part was not the metaphor itself, but the shared model it carried.

Upstream and downstream are the same kind of tool. They are river words, but a river metaphor only helps if everyone knows which river they are standing in.

## The Case

Imagine a listing page in a marketplace.

It renders a set of items. For each item, the page needs display metadata, availability, inspection status, price, shipping promise, and whether the item is eligible for a promotion. The page itself is not the source of truth for most of that, so it calls several services:

```text
listing-page
  -> catalog
  -> pricing
  -> inventory
  -> promotion
  -> shipping-promise
```

One afternoon, some items show the wrong delivery promise. They appear eligible for next-day delivery even though inspection has not completed.

During triage, someone says:

> "The downstream service sent us the wrong status."

That sounds clear enough. The listing page called another service, and that service returned bad data.

A few minutes later, though, the investigation gets awkward. The shipping-promise service did not invent the status. It read inspection state from another feed, and that state was delayed because the warehouse system had not finished processing the item. The display page was showing a value derived from a chain of facts, not a fact owned by the service it called.

Now the team has at least three directions in play.

```text
Call flow:
listing-page -> shipping-promise -> inspection-feed

Data flow:
warehouse -> inspection-feed -> shipping-promise -> listing-page

Business flow:
discovery -> display -> checkout -> payment -> fulfillment
```

Which one is upstream?

The answer depends on the noun you forgot to say.

## Upstream of What?

I used to treat upstream and downstream as obvious words. They are not. They only become useful after you name the flow, or less formally, after you name the river.

If you mean call direction, the service you call is downstream from your caller. If the listing page calls pricing, pricing is a downstream dependency of the listing page.

If you mean data origin, the producer is upstream of the data consumer. If pricing owns the price, pricing is upstream of the listing page for that field.

If you mean failure propagation, the failing dependency may be upstream in the causal chain even if it is downstream in the call graph.

If you mean business flow, the product listing may be upstream of checkout, and checkout may be upstream of fulfillment.

If you mean state ownership, the source of truth is upstream of every derived view.

All of those uses are defensible.

That is the problem.

## The Word Hides the Axis

The dangerous part is not that engineers use different flows. That happens. The dangerous part is that the same word can make people feel aligned before they actually are.

In the listing incident, "downstream service" sounded precise. It was not. It collapsed several questions into one phrase:

- Which service did we call?
- Which system owned the incorrect field?
- Which system derived it?
- Which workflow step consumed it?
- Which dependency could fail independently?
- Which boundary should validate it?

Those are different questions, and they deserve different answers.

When a design review says "the downstream service should handle this," the same ambiguity appears. Does downstream mean the callee? The data owner? The next step in the workflow? The consumer of an event? The system that suffers if this breaks?

Without the axis, upstream and downstream are not architecture words.

They are vibes.

## A Better Habit

I do not think teams need to ban the words.

They are useful shorthand once the map is clear. The problem is using them before the map is clear.

The safer habit is to say the flow first.

Instead of:

```text
The downstream service returned bad data.
```

Say:

```text
In the call flow, listing-page called shipping-promise.
Shipping-promise returned a delivery promise derived from inspection state.
Inspection is the source of truth for that state.
```

That is longer.

It is also harder to misunderstand.

For design reviews, I prefer to separate the axes explicitly:

```text
Call flow:
listing-page -> shipping-promise

Data flow:
inspection owns inspection_status
shipping-promise derives delivery_promise
listing-page renders delivery_promise

Failure flow:
shipping-promise timeout degrades listing-page
stale inspection state can produce an incorrect promise

Validation boundary:
shipping-promise must not promise next-day delivery unless inspection_status is complete
listing-page must treat missing promise as unavailable, not eligible
```

Now the discussion has shape.

The team can still say "downstream" after that, but the word points to a named flow instead of replacing one.

## This Shows Up Everywhere

This is not limited to service calls.

In event-driven systems, teams often say a downstream consumer failed. That usually means "a consumer of this topic failed." But the consumer may own the next durable state in the business flow.

In data pipelines, upstream often means the raw source table. But for an application team, the feature store or derived table may be the dependency they experience as upstream.

In commerce systems, a display page can be upstream of checkout in the business flow, downstream of catalog in the call flow, and downstream of pricing in the data flow.

None of these are wrong.

They are just different rivers.

## What I Try to Say Now

When the conversation matters, I try to avoid naked upstream and downstream.

I say:

- "the service we call"
- "the service that calls us"
- "the source of truth for this field"
- "the derived view"
- "the producer of this event"
- "the consumer of this event"
- "the next step in the workflow"
- "the dependency that can break this path"

They are less elegant, but they force the axis into the sentence.

The point is not vocabulary purity. The point is to keep system conversations attached to the thing they are actually about.

Calls, data, failures, ownership, and workflows all have direction. They are just not always the same direction, which is why upstream and downstream are useful only after you name the flow.

If you do not name the flow, you are not describing the system.

You are asking everyone in the room to guess which river you meant.
