# System Design Refresher

A plain English guide to system design for senior software engineers. It's meant to help you brush up on the fundamentals and catch up on what changed in the last few years.

The main doc is [system-design-guide.md](./system-design-guide.md).

## Who this is for

- Senior engineers getting ready for system design interviews
- Engineers about to lead or review a big design
- Anyone who has been heads down on one system for a while and wants to know what's current

It assumes you already write software for a living. It doesn't explain what a database is. It explains when to pick which one and what it costs you.

## What's inside

| Section | What it covers |
|---|---|
| 1. How to Use This Doc | Where to start depending on what you need |
| 2. How to Approach Any Design | A step by step process for interviews and real reviews |
| 3. Back of the Envelope Math | Latency numbers, conversions, a worked estimate |
| 4. Core Building Blocks | DNS, CDN, load balancers, gateways, service mesh, caching, object storage, search |
| 5. Data Storage | Picking a database, SQL vs NoSQL, indexes, B-tree vs LSM |
| 6. Replication, Partitioning and Consistency | Replication types, sharding, CAP/PACELC, consistency models, Raft, clocks, CRDTs |
| 7. Messaging, Queues and Streams | Queues vs logs, delivery guarantees, ordering, DLQs |
| 8. Distributed Transactions and Workflows | 2PC, sagas, outbox, CDC, durable execution, idempotency |
| 9. Reliability and Resilience | SLOs, timeouts, retries, circuit breakers, rate limiting, multi region, cells, DR |
| 10. API Design | REST, gRPC, GraphQL, pagination, real time options, webhooks |
| 11. Observability | Metrics, logs, traces, profiles, OpenTelemetry, alerting |
| 12. Security | Authn, authz, encryption, zero trust, multi tenancy, supply chain |
| 13. Data Platforms and Analytics | OLTP vs OLAP, batch vs stream, lakehouse, Iceberg, DuckDB |
| 14. What Is Current in 2026 | LLM systems, RAG, agents, Postgres everywhere, cost, platform engineering, and more |
| 15. Worked Examples | URL shortener, rate limiter, feed, chat, notifications, scheduler, LLM support assistant |
| 16. Trade-off Cheat Sheet | Common decisions side by side |
| 17. Common Mistakes | What goes wrong in interviews and real reviews |
| 18. How to Stay Current | Books, papers, blogs, habits |
| 19. Final Checklist | Run through before you call a design done |

## How to use it

**Interview in a few days?** Read sections 2, 3, 15, 16 and 19. Then go back to whatever felt shaky.

**Design review coming up?** Start with section 2, then jump to whatever matches your problem. Use section 19 as a checklist before the meeting.

**Just staying sharp?** Read section 14 and pick something from section 18.

## Keeping it up to date

This stuff moves. Section 14 especially, since AI infrastructure is changing fast.

- Review the doc every 6 months or so
- Update section 14 with anything that became standard (or stopped being relevant)
- Update the "Last updated" date at the top of the guide
- If a tool or product named in the doc gets deprecated, renamed or relicensed, fix it

The fundamentals in sections 2 through 9 don't change much. The tools do.

## A note on tone

This is written the way an engineer would explain it to another engineer. Plain words, short sentences, no fluff. If something reads like marketing, it should probably be cut.
