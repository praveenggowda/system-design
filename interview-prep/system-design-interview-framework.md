# System Design Interview Framework

## The Book vs Reality

Alex Xu's framework is a starting point, not the finish line.
The book teaches you how to structure an answer.
What gets you hired is engineering reasoning, not a perfect diagram.

Interviewers at senior levels have said explicitly:
they would rather hear why you chose one approach over another
and what you are giving up
than watch you draw a textbook architecture.

## The Framework (Refined)

### Step 1: Understand the Problem (5 to 10 minutes)

Do not assume. Do not start designing.

Ask:
- What are we building? What is in scope and what is out of scope?
- Who are the users? What do they actually need?
- What is the scale? Users today, in 6 months, in a year?
- What are the consistency requirements? Is eventual consistency acceptable?
- What are the latency requirements? Real-time or batch?
- What happens when it fails? Is availability more important than consistency?
- What exists already? What can I reuse?

Why this matters:
A notification system for 1,000 users is a monolith with a cron job.
A notification system for 10M users is Kafka, workers, and a delivery layer.
The question you skip here changes the entire design.

Write down your assumptions. State them out loud. Get confirmation before you draw anything.

### Step 2: Define Scope and Constraints (2 to 5 minutes)

Before drawing anything, align on numbers.

Estimate:
- Daily active users
- Requests per second (reads and writes separately)
- Data storage over 5 years
- Bandwidth

State your non-functional requirements:
- Availability target (99.9%, 99.99%)
- Latency target (p99 under 200ms?)
- Consistency model (strong, eventual, causal)
- Durability (can we lose data?)

This step separates senior engineers from junior ones.
Junior engineers skip to the diagram.
Senior engineers know the numbers drive every architectural decision.

### Step 3: High Level Design and Get Buy-In (10 to 15 minutes)

Draw the simplest design that works.

Start with:
- Client
- API layer
- Core service(s)
- Database
- Any async components

Think out loud. Do not draw silently.
Say: "I am going to put a queue here because of X. Does that make sense to you?"

Get buy-in before going deep.
If you go deep on the wrong part you waste the interview.
Ask: "Which part would you like me to dig into?"

### Step 4: Design Deep Dive (10 to 25 minutes)

Pick the two or three hardest problems in the system and show depth.

Common deep dive areas:
- Database schema and query patterns
- How you handle scale (sharding, partitioning, replication)
- How you handle failures (retries, idempotency, dead letter queues)
- How you handle consistency (two-phase commit, sagas, outbox pattern)
- Caching strategy (what to cache, eviction policy, invalidation)
- API design (REST vs event-driven, pagination, rate limiting)

Do not try to cover everything. Cover the hardest parts well.

The whiteboard diagram gets you to "meets expectations."
The trade-off reasoning gets you to "strong hire."

### Step 5: Wrap Up (3 to 5 minutes)

This step is almost always ignored. Use it.

Cover:
- What you would do differently with more time
- What the system cannot handle yet (and what would break first)
- What breaks at 10x scale
- Monitoring and observability (how do you know the system is healthy?)
- Operational concerns (how do you deploy and roll back safely?)

This shows production thinking. Any engineer can draw a diagram.
Senior engineers know what happens after deployment.

## What Good Looks Like vs What Bad Looks Like

| Bad | Good |
|---|---|
| Starts drawing immediately | Asks clarifying questions first |
| Silent while drawing | Thinks out loud |
| States one solution | Presents options with trade-offs |
| Covers everything shallowly | Covers hard parts deeply |
| Never mentions failure | Designs for failure explicitly |
| Skips wrap-up | Identifies what breaks and what is missing |
| Memorised patterns | Adapts to the specific problem |

## The One Rule

Trade-offs matter more than correctness.

There is no perfect design. Every choice gives up something.
The interviewer wants to know you understand what you are giving up.

"I chose Cassandra over PostgreSQL here because we need write throughput
at this scale and we can tolerate eventual consistency on reads.
The trade-off is we lose strong consistency and transactions,
which means this part of the system needs compensating logic."

That sentence is worth more than the entire diagram.

## For Financial Systems

Core banking and financial platforms have different priorities than consumer internet systems:

- Correctness over availability (money cannot be lost)
- Event-driven systems and event sourcing
- Auditability (every state change must be traceable)
- Financial correctness (double-entry, idempotency, ACID)

When designing any financial system, default to consistency first, availability second.
That is the opposite of most consumer internet systems.

## Key Questions to Always Ask

1. What is the read/write ratio?
2. What is the consistency requirement?
3. What happens if a component fails?
4. How do we know the system is healthy? (observability)
5. What does 10x scale break first?

## Takeaway from Alex Xu

The framework is correct as a skeleton.
The book is good for learning patterns (URL shortener, rate limiter, notification system).
It is not sufficient on its own for senior roles.

Supplement with:
- Designing Data-Intensive Applications by Martin Kleppmann (for depth on distributed systems)
- Real-world incident post-mortems (for production thinking)
- Mock interviews with feedback (diagrams do not teach you to think out loud)

Sources: Alex Xu System Design Interview Vol 1 and Vol 2, interviewing.io senior engineer guide, Pragmatic Engineer, ByteByteGo.
