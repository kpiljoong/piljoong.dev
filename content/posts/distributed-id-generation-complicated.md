+++
date = '2025-11-27T11:16:56+09:00'
draft = false
title = "Distributed ID Generation: The Simple Thing That Gets Complicated Fast"
description = "Why ID formats become long-term architectural commitments, and how UUID, ULID, Snowflake, and custom schemes compare."
tags = ["distributed-systems", "ids", "architecture", "design"]
categories = ["backend", "engineering", "architecture"]
slug = "distributed-id-generation-complicated"
+++

Most systems start with auto-increment IDs because it's the easiest possible thing that works. The database hands you numbers, you store them, life is good. There's something comforting about watching IDs tick upward in perfect sequence—12345, 12346, 12347.

But IDs have a funny property: they quietly spread everywhere. Into URLs, logs, analytics pipelines, API responses, customer support workflows—all the places you don't think about until changing the format suddenly becomes painful.

Distributed IDs are a known problem at scale. But I didn't expect the format to matter as much as it does. After watching teams struggle with migrations and evolution, I realized the real problem isn't generation—it's choosing a format you can live with long-term.

## The First Time IDs Became a Problem

The first time I saw ID formats become a real architectural constraint was during a database split. A team was breaking a monolith into several database instances. The old auto-increment IDs were totally fine—until suddenly they weren't, because multiple shards couldn't share the same global counter anymore.

The migration itself wasn't terrible. The ugly part was everything else. IDs already existed in URLs, references in other services, analytics jobs expecting sequential integers, dashboards that assumed ordering. You can't just regenerate everything because the IDs already have meaning out in the world.

Their workaround was simple and surprisingly effective: they offset new IDs by a huge constant—roughly a billion. Old IDs stayed below the threshold, new IDs lived above it, and nothing collided. It worked surprisingly well, but it also taught me something.

**ID formats aren't just formats. They're commitments.**

Once you deploy one, it becomes part of your architecture. That realization stuck with me.

## When Does This Actually Matter?

If your system is running on a single database with moderate traffic, auto-increment is still probably the best answer. Don't overthink it.

But there are a few inflection points that force you into distributed ID generation whether you want it or not. Sharding is the classic one. The moment you split your database, the "just increment a number" trick stops working. Multi-region deployments are another—clocks drift, network latency isn't stable, and depending on another region to hand you an ID is both slow and fragile.

Microservices make it worse. One service generating IDs is fine. Ten services doing it means you accidentally built a distributed system, and centralizing ID generation sounds elegant on a whiteboard but in practice becomes slow and tightly coupled.

Then there are offline-first systems—mobile, IoT, edge nodes. If something needs to create objects when it's offline, IDs must be generated locally. And if you're building event-driven systems with Kafka or similar, time-ordered IDs simplify querying and debugging in ways that random UUIDs just don't.

When any of these show up, you're not just choosing an ID format anymore—you're inheriting a distributed systems problem.

## What People Usually Reach For

Once you go beyond auto-increment, most people try one of three things.

**UUIDv4 is the simplest option.** You get uniqueness without coordination, which is great. But you lose everything else—no ordering, no structure, terrible index locality. It works when your problem is genuinely just "give me a unique value," but it falls short pretty quickly if you care about anything beyond that.

**ULID and UUIDv7 are where a lot of people land.** Timestamp-based IDs offer time ordering without needing a central coordinator. I like ULID a lot—it's simple, portable, and genuinely useful. But once you hit high concurrency or multi-region setups, some cracks appear.

You can generate thousands of IDs in the same millisecond, and ordering becomes fuzzy. ULID offers per-process best-effort monotonicity only if the implementation adds a monotonic counter. Many libraries do, but the spec doesn't require it. Each process only knows about its own IDs, not what other nodes generated in the same millisecond. True distributed monotonicity is impossible without coordination.

Clock drift is the other issue. If system time moves backward, ULID ordering breaks. I've seen this create really confusing bugs where yesterday's events suddenly sort after today's in an analytics pipeline. Different libraries handle this differently, but there's no standard recovery mechanism.

The format also has no space for metadata. You can't encode tenant information, shard hints, or type data. It's purely timestamp plus randomness, which is fine until you need more structure.

UUIDv7 cleans up some of ULID's specification ambiguities, but it inherits the same core limitations.

**Snowflake gives you strong ordering and performance.** It divides bits into fixed fields—timestamp, worker ID, sequence counter. For the right use case, this is excellent. You get strong per-node monotonicity and huge throughput.

But Snowflake's rigidity becomes a constraint over time. Worker IDs need coordination, which gets harder with autoscaling. Bit allocations limit future flexibility—I've seen teams outgrow their initial worker bit layout and realize there's no graceful way to evolve. You either version the format (now you have two ID systems in production) or migrate (expensive).

## Two Dimensions That Matter

After working with these formats for a while, I started thinking about ID design along two axes.

The first is ordering guarantees. Some formats give you nothing (UUIDv4), some give you approximate ordering that's good enough most of the time (ULID), and some give you strict per-node ordering (Snowflake). What you need depends on how much you care about events being sortable and whether "roughly ordered" is acceptable.

The second is structure and extensibility. Some formats are completely opaque—just bits that happen to be unique (UUIDv4). Some have fixed bit allocations that you can't change later (Snowflake). And some let you encode additional metadata like tenant IDs or shard hints (that's where I ended up with OrderlyID).

## Why I Built OrderlyID

After dealing with these tradeoffs repeatedly, I wanted something specific: structured but not rigid, typed but still human-friendly, evolvable instead of locked in forever, decentralized but sortable, and suitable for multi-tenant systems.

The typed prefix idea was popularized by TypeID, and it's genuinely useful. When you see `order_01h8n6qj...` vs `user_01h8n6qj...` in logs, you immediately know what you're looking at. It seems like a small thing, but it prevents entire classes of bugs where IDs get mixed up across services.

But I wanted the payload itself to be structured too. So OrderlyID uses a 160-bit layout:

```text
┌─────────────────────────────────────────────────────────────────┐
│                    OrderlyID Bit Layout (160 bits)              │
├──────────┬────────┬──────────┬──────────┬──────────┬────────────┤
│ 48 bits  │ 8 bits │ 16 bits  │ 12 bits  │ 16 bits  │  60 bits   │
│timestamp │ flags  │  tenant  │ sequence │  shard   │  random    │
└──────────┴────────┴──────────┴──────────┴──────────┴────────────┘
```

The timestamp gives you time ordering, same as ULID. The sequence counter (12 bits) handles up to 4,096 IDs per millisecond per process, so you get proper monotonicity within that window. The tenant field (16 bits) can encode up to 65,535 different tenants, and the shard field does the same for routing hints.

The flags field includes version bits for evolution and a privacy flag for time bucketing—useful when you don't want IDs to leak precise generation timestamps in user-facing contexts.

And there's an optional checksum. I initially thought nobody would use it. Then I watched an admin type an ID incorrectly three times in a row during a support call. Turns out checksums catch real mistakes.

Here's what it looks like in practice:

```text
order_01h8n6qj3k9m2p4r6s8t0v2w4x6y8z0a
user_01h8n6qj3k9m2p4r6s8t0v2w4x6y8z0b
payment_01h8n6qj3k9m2p4r6s8t0v2w4x6y8z0c-a1b2
```

The tenant field turned out to be more useful than I expected. Encoding tenant IDs directly in the identifier means the database can route or filter by ID prefix alone—no table scans, no complex joins, just straight prefix matching. In multi-tenant systems at scale, this actually matters.

**OrderlyID makes different tradeoffs than ULID or Snowflake.** It's longer (32+ characters vs ULID's 26), more complex to implement, and not a standard. It still depends on system clocks like every other timestamp-based format, so clock drift affects it the same way.

But it gives you structure that ULID doesn't, evolution paths that Snowflake doesn't, and type safety that both lack. It works well for systems where those things matter. If they don't, use something simpler.

## How I Think About This Now

When I'm choosing an ID format today, I think about it like this:

If you've got a single database and no plans to shard, use auto-increment. If you just need uniqueness and don't care about ordering, UUIDv4 is fine. If you want time ordering without coordination and moderate concurrency is good enough, go with ULID or UUIDv7.

If you need very high throughput and strict per-node ordering, and you're willing to manage worker IDs, Snowflake makes sense. And if you need structured fields, tenant routing, type safety, or an evolution path, something like OrderlyID might fit better.

It's not about "best"—it's about what constraints you're actually dealing with.

## Things That Surprised Me

A few things about OrderlyID turned out differently than I expected.

I thought about using hybrid logical clocks to handle clock drift better, but the complexity felt disproportionate. If you need true causal ordering across distributed nodes, you probably want CRDTs or vector clocks anyway—IDs aren't the right layer for that.

I expected typed prefixes to feel heavy and unnecessary. They didn't. Seeing `order_...` vs `user_...` in logs turned out to be way more valuable than I thought, especially in multi-service environments where type confusion is a recurring pain point.

And I thought checksums would be purely theoretical. Then reality showed up. Manual workflows, admin consoles, spreadsheets—people mistype IDs more often than you'd expect.

## What This Really Comes Down To

ID generation looks trivial until you have to change your ID format. Then it becomes one of the most painful migrations you'll deal with.

The trick is choosing something that fits your system now and still fits it later. Auto-increment is great until it isn't. ULID is great until you need structure. Snowflake is great until you outgrow the layout.

OrderlyID is just another point in that design space—one that matches needs I kept running into. 

ID formats look like tiny details, but they become part of the foundation of your system. Choose one that fits your constraints today, but won't trap you tomorrow.

The spec and implementations are here if you want to look:

- https://github.com/orderlykit/orderlyid
