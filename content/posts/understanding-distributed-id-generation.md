+++
date = '2025-11-27T11:16:56+09:00'
draft = true
title = "Understanding Distributed ID Generation: When You Need It, Why It's Hard, and What the Tradeoffs Look Like"
+++

# Understanding Distributed ID Generation: When You Need It, Why It's Hard, and What the Tradeoffs Look Like

Most systems start with auto-increment IDs, and that's fine. They're simple, predictable, and work perfectly well until they don't. The problem is that "until they don't" usually arrives without much warning, often when you're already dealing with other scaling challenges.

I've seen this play out a few times now. The transition from simple sequential IDs to distributed generation is never just a technical change—it ripples through your entire architecture in ways that aren't obvious until you're in the middle of it.

## How I Got Here

A few years back, I watched a team migrate from sequential database IDs to Snowflake-based distributed IDs. Nothing was broken—the auto-increment system worked fine. But they were splitting their monolithic database to handle growth, and suddenly IDs needed to be generated across multiple independent instances.

The migration itself was straightforward enough technically. The messier part was dealing with the existing IDs. They couldn't just regenerate everything—these IDs were already in URLs, external APIs, customer-facing systems, and analytics pipelines. Their solution was to offset the new IDs by a large constant (something like a billion), keeping old and new ID spaces separate through simple arithmetic.

It worked, but it highlighted something I've been thinking about since: ID formats aren't just about generation—they're about evolution. Once you've deployed an ID format, it gets embedded everywhere. Changing it later isn't impossible, but it's expensive in ways that compound over time.

## When Auto-Increment Actually Becomes a Problem

Let me be direct: if you're running a single database with moderate write traffic, you probably don't need to think about distributed IDs. I've seen too many teams overengineer this early.

But there are real triggers that make distributed ID generation necessary rather than just interesting:

**Database sharding** changes the game. You can't coordinate a global sequence across shards without creating exactly the bottleneck you were trying to avoid. Each shard needs to generate IDs independently.

**Multi-region deployments** introduce clock skew and make centralized coordination impractical. Local ID generation becomes necessary for both latency and availability.

**Microservices architectures** multiply the problem. When you've got dozens of services all generating identifiers, forcing them through a central ID service creates coupling you don't want and a single point of failure you definitely don't want.

**Offline-first or edge computing** scenarios make it impossible to talk to a central database for ID generation. Mobile apps, IoT devices, and edge deployments need to generate valid IDs locally.

**Event-driven systems** often benefit from time-ordered IDs. When you're processing streams of events through Kafka or similar systems, having IDs that sort lexicographically by time simplifies a lot of downstream logic.

If you're not hitting these patterns, stick with what works. There's no prize for complexity.

## The Landscape of Distributed IDs

Most distributed ID approaches fall into three broad categories, each making different tradeoffs. But before driving into specifics, it's useful to understand the two core dimensions that separate these approaches:

**Dimension 1: Ordering guarantees**
- None (random UUIDs)
- Approximate (timestamp-based, best-effort across machines)
- Strict per-node (structured formats with sequence counters)

**Dimension 2: Structure and extensibility**
- Opaque (pure randomness or timestamp+random)
- Semi-structured (fixed bit allocations)
- Fully structured (typed prefixes, extensible fields)

Different systems need different points on these two axes. There's no universally correct choice—just tradeoffs aligned with your constraints.

Timestamp-Based IDs (ULID, UUIDv7)
These are probably the most straightforward conceptually. Take a timestamp, add some randomness, encode it. ULID is a good example: 48 bits of millisecond timestamp, 80 bits of randomness, Base32 encoded to 26 characters.

The appeal is obvious—you get approximate time ordering without any coordination. Generate them anywhere, and they'll mostly sort by creation time. For a lot of systems, "mostly" is good enough.

The catches are subtle but real. Millisecond timestamps mean that under high concurrency, you're generating multiple IDs in the same millisecond, and their ordering becomes ambiguous. Many ULID implementations add monotonic counters to handle this, but the behavior isn't standardized.

Clock drift is the other issue. These formats assume your system clocks are reasonably accurate. In practice, NTP adjustments happen, virtualized environments drift, and multi-region deployments have synchronization quirks. If a clock jumps backward, you can generate IDs that sort before IDs you generated moments ago.

ULID also doesn't give you structured fields. It's timestamp plus randomness, nothing more. If you need to encode tenant information, shard hints, or type data in the ID itself, you're out of luck.

Structured IDs (Snowflake, Sonyflake)
Twitter's Snowflake took a different approach: explicit bit allocation for time, worker ID, and sequence counter. You get strong monotonicity per worker and high throughput—4,096 IDs per millisecond per worker.

The structure is both the strength and the limitation. Allocating 10 bits for worker IDs seemed reasonable until you need more than 1,024 workers. The bit layout is rigid, and evolving it means versioning your entire ID format.

You also need to manage worker ID assignment, which adds operational complexity. It's not huge, but it's something you didn't have to think about with timestamp-based approaches.

Random IDs (UUID v4)
Sometimes you just want globally unique IDs and don't care about ordering. UUID v4 does this well—purely random, no coordination needed, vanishingly small collision probability.

The downside is exactly what you'd expect: no time ordering, poor index locality, and they're not particularly useful for anything beyond "unique identifier." If you're building event systems or need to partition data by time, random IDs don't help much.

## What I Learned from ULID at Scale

ULID has become really popular, and for good reason. It's simple, portable, and works well for most distributed systems. But I've run into its limitations enough times to have opinions about where it falls short.

The monotonicity story gets fuzzy under load. If you're generating thousands of IDs per second, you'll have many IDs in each millisecond bucket. Some libraries handle this with counters, others don't. The spec doesn't mandate behavior here, so you get implementation-dependent ordering guarantees.

More fundamentally, ULID provides per-process monotonicity at best. True distributed monotonicity across multiple generators is impossible without coordination—each process only knows about its own IDs, not what other processes generated in the same millisecond.

Clock drift is something I've seen bite people in multi-region deployments. It's not catastrophic—we're usually talking about small discrepancies—but when you're debugging ordering issues in a distributed system, finding out that clock skew caused your IDs to sort unexpectedly is frustrating.

The lack of structure has been the bigger issue for systems I've worked on. In multi-tenant architectures, encoding the tenant ID in the identifier itself is useful for routing, partitioning, and query optimization. ULID doesn't give you space for that. Same with shard hints or type information—if you want it, you need to manage it separately.

I should mention UUIDv7 here. It's a more recent spec that addresses some of ULID's ambiguities, particularly around monotonicity. It's a solid evolution of timestamp-based IDs. But it still has the clock dependency and limited extensibility of the format.

## Where Snowflake Shows Its Age

Snowflake-style IDs work well when your requirements fit the format. Strong per-node monotonicity, high throughput, predictable bit layout—these are real advantages.

But that rigidity becomes a constraint over time. I've seen systems outgrow their initial worker bit allocation. I've seen teams want to add region information to the ID and realize there's no space for it. Evolving a Snowflake format means either versioning (now you have two ID formats in production) or migrating (expensive).

The operational overhead of managing worker IDs isn't huge, but it's another thing to coordinate. In cloud environments with autoscaling, you need to think about worker ID assignment and reclamation.

## Why I Built OrderlyID

After working with these systems for a while, I wanted something that balanced structure with flexibility. OrderlyID is my attempt at finding a different point in the design space.

The core idea is typed, structured IDs that can evolve without format changes:
```
order_01h8n6qj3k9m2p4r6s8t0v2w4x6y8z0a
user_01h8n6qj3k9m2p4r6s8t0v2w4x6y8z0b
```

The prefix provides type safety. It's harder to accidentally use an order ID where you need a user ID, and when debugging, you know what you're looking at immediately.

The payload encodes structure:
```
[48b: timestamp] [8b: flags] [16b: tenant] [12b: sequence]
[16b: shard] [60b: random]
```

This gives you:
- Time ordering (with the same clock dependency as ULID)
- Per-process monotonicity via the sequence counter
- Tenant and shard fields for routing
- Flag bits for versioning and feature flags
- Enough entropy to avoid collisions

The tenant field has a practical benefit beyond just metadata: encoding tenant IDs directly in the identifier can reduce query fanout in multi-tenant systems. The database can route or filter by ID prefix alone, avoiding table scans or complex tenant lookups.

The optional checksum catches transcription errors, which matters if IDs ever get typed or spoken.

I'm not claiming OrderlyID is universally better than ULID or Snowflake. It makes different tradeoffs:

**Tradeoffs I accepted**:
- Still depends on system clocks (like ULID, UUIDv7)
- Longer than Snowflake (26+ characters vs 18-19)
- More complex to implement than ULID
- Not a standard like UUID

**What I prioritized**:
- Type safety through prefixes
- Structured fields without centralized coordination
- Evolution path through flag bits
- Multi-tenant and sharding use cases

OrderlyID works for systems where type confusion is a real bug source, where you need tenant or shard routing in the ID, and where you want to evolve the format over time. If those aren't your concerns, ULID or UUIDv7 are probably simpler choices.

## Choosing an ID Strategy

Here's how I think about this now:

**Stick with auto-increment** if you're on a single database and not planning to shard soon. It's simple, it works, and premature distribution is real.

**Use UUID v4** if ordering doesn't matter and you want maximum simplicity. Good for truly random identifiers where time isn't relevant.

**Consider ULID or UUIDv7** if you want time ordering without coordination overhead. Works well for most distributed systems with moderate concurrency. UUIDv7 is more standardized, ULID has better library support currently.

**Use Snowflake-style IDs** if you need strict per-node monotonicity and high throughput, and you're okay with managing worker IDs. Good choice if you control your infrastructure and don't expect the format to change.

**Look at structured formats** (like OrderlyID) if you need explicit tenant/shard fields, type safety matters in your domain, or you want an evolution path built in. More complexity upfront but can save migration pain later.

The meta-lesson I've learned: ID design matters more when you need to change it. Initial generation is usually the easy part. It's the evolution—adding fields, changing formats, migrating existing IDs—that's expensive. If you can anticipate those needs, factor them into your decision. If you can't, start simple and be ready to migrate when the time comes.

## What I Got Wrong

I considered using hybrid logical clocks for OrderlyID to handle clock drift better. I decided against it because the complexity seemed disproportionate to the benefit for most use cases. If you need true causal ordering across distributed nodes, you probably want CRDTs or vector clocks anyway, not just smarter IDs.

I also initially thought the prefix typing would feel heavyweight. It hasn't—turns out seeing `order_...` vs `user_...` in logs and debugging is more useful than I expected.

The checksum remains optional because not everyone wants the extra characters, but I've found it catches enough typos in operations work to be worth it.

## Closing Thoughts

ID generation is one of those problems that seems simple until you're actually dealing with it at scale. The technical challenges—monotonicity, clock drift, collision avoidance—are well understood. The hard part is usually the evolution story: how do you change your ID format three years from now when requirements have shifted?

I don't think there's one right answer. Auto-increment is great until it's not. ULID is simple until you need structure. Snowflake is powerful until you hit its limits. OrderlyID is my attempt to explore a different set of tradeoffs, but it's not universal either.

The spec and implementations for OrderlyID are on GitHub if you're interested in the details or want to experiment with the approach. I'm particularly interested in hearing from people dealing with multi-tenant systems or anyone who's gone through painful ID migrations—those stories tend to be the most educational.

**Links:**
- [OrderlyID Specification](https://github.com/orderlykit/orderlyid)
- [Example implementations](https://github.com/orderlykit)
