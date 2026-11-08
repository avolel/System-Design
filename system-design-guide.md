# System Design for Senior Engineers

A refresher and a field guide. Use it before an interview, before a big design review, or when you just want to check that what you know is still current.

Last updated: November 2026

---

## Table of Contents

1. [How to Use This Doc](#1-how-to-use-this-doc)
2. [How to Approach Any Design](#2-how-to-approach-any-design)
3. [Back of the Envelope Math](#3-back-of-the-envelope-math)
4. [Core Building Blocks](#4-core-building-blocks)
5. [Data Storage](#5-data-storage)
6. [Replication, Partitioning and Consistency](#6-replication-partitioning-and-consistency)
7. [Messaging, Queues and Streams](#7-messaging-queues-and-streams)
8. [Distributed Transactions and Workflows](#8-distributed-transactions-and-workflows)
9. [Reliability and Resilience](#9-reliability-and-resilience)
10. [API Design](#10-api-design)
11. [Observability](#11-observability)
12. [Security](#12-security)
13. [Data Platforms and Analytics](#13-data-platforms-and-analytics)
14. [What Is Current in 2026](#14-what-is-current-in-2026)
15. [Worked Examples](#15-worked-examples)
16. [Trade-off Cheat Sheet](#16-trade-off-cheat-sheet)
17. [Common Mistakes](#17-common-mistakes)
18. [How to Stay Current](#18-how-to-stay-current)
19. [Final Checklist](#19-final-checklist)

---

## 1. How to Use This Doc

You already know most of this. The point is to refresh it, fill gaps, and catch the stuff that changed since you last looked.

- If you have an interview in a few days, read sections 2, 3, 15, 16 and 19 first. Then go back to whatever felt weak.
- If you are preparing for a real design review, read section 2, then jump to the sections that match your problem.
- If you just want to stay current, read section 14 and section 18.

Each section is written so you can read it on its own. There is some repetition on purpose.

One thing to keep in mind the whole time. At the senior level nobody cares if you can name every component. They care if you can explain why you picked one thing over another, what it costs you, and what breaks first.

---

## 2. How to Approach Any Design

This works for interviews and for real work. The steps are the same, the time you spend on each one is different.

### Step 1. Clarify the problem

Don't start drawing boxes. Ask questions first.

- Who are the users and what are they trying to do?
- What are the core features? Pick 2 or 3. Say out loud what you are leaving out.
- What scale are we talking about? Users, requests per second, data size, growth.
- What are the read and write patterns? Read heavy, write heavy, bursty?
- What matters most: latency, consistency, availability, cost?
- Any hard constraints? Regulations, regions, existing systems, team size.

Write the answers down. You will refer back to them.

### Step 2. Define requirements

Split them in two.

**Functional requirements.** What the system does. "Users can post a message." "Users can see a feed of people they follow."

**Non-functional requirements.** How well it does it. "p99 read latency under 200ms." "99.9% availability." "No data loss once a write is acknowledged." "Works in US and EU, EU data stays in EU."

Non-functional requirements drive most of the design. A feed that can be a few seconds stale is a very different system from a bank ledger.

### Step 3. Estimate

Do quick math on traffic, storage and bandwidth. See section 3. You are not trying to be exact. You are trying to find out if this fits on one machine, a few machines, or needs real partitioning.

### Step 4. Sketch the API and data model

Write the main API calls. Write the main entities and how they relate. This forces you to understand the problem before you pick technology.

```
POST /v1/messages         { channel_id, body, idempotency_key }
GET  /v1/channels/{id}/messages?cursor=...&limit=50
```

```
Message(id, channel_id, author_id, body, created_at)
Channel(id, name, created_at)
Membership(user_id, channel_id, role, joined_at)
```

### Step 5. High level design

Now draw the boxes. Client, edge, load balancer, services, data stores, queues, caches. Walk through one request end to end. Then walk through a write end to end.

Keep it simple at first. One service and one database is a fine starting point if the numbers allow it.

### Step 6. Deep dive

Pick the 2 or 3 hardest parts and go deep. Usually these are:

- the hot path (the request that happens the most)
- the part with the hardest consistency needs
- the part that will hit scale limits first

### Step 7. Talk about failure and operations

What happens when a node dies, a region goes down, a dependency gets slow, a deploy goes bad? How do you know something is wrong? How do you roll back?

This is the part that separates senior from mid level. Spend real time on it.

### Step 8. Wrap up with trade-offs

Say what you would do differently with more time, more money, or 10x the traffic. Name the weak spots in your own design before someone else does.

---

## 3. Back of the Envelope Math

### Numbers worth remembering

These are rough. Hardware changes, but the ratios stay about the same.

| Operation | Rough time |
|---|---|
| L1 cache reference | ~1 ns |
| Main memory reference | ~100 ns |
| Compress 1 KB with a fast codec | ~2 µs |
| Read 1 MB sequentially from memory | ~5 to 10 µs |
| Random read from NVMe SSD | ~20 to 100 µs |
| Round trip inside same datacenter / AZ | ~0.5 ms |
| Round trip between AZs in same region | ~1 to 2 ms |
| Read 1 MB sequentially from SSD | ~50 to 200 µs |
| Disk seek on spinning HDD | ~5 to 10 ms |
| Round trip US East to US West | ~60 to 70 ms |
| Round trip US to Europe | ~80 to 100 ms |
| Round trip US to Asia/Australia | ~150 to 200 ms |

What to take from this:

- Memory is way faster than network. Network inside a datacenter is way faster than across the world.
- Every cross region call in the hot path costs you real latency. Avoid them for user facing reads.
- A single SSD can do hundreds of thousands of IOPS. A lot of systems are not as big as people think.

### Handy conversions

- 1 day is about 86,400 seconds. Round to 100,000 for quick math.
- 1 million requests per day is about 12 requests per second.
- 100 million requests per day is about 1,200 requests per second.
- 1 billion requests per day is about 12,000 requests per second.
- Peak traffic is usually 2x to 5x the average. Some products (sports, ticketing, shopping events) spike 10x or more.

### Rough capacity of common things

These depend a lot on workload, so treat them as "order of magnitude".

- One well tuned app server: thousands to tens of thousands of simple requests per second.
- One Postgres or MySQL primary on good hardware: thousands to tens of thousands of writes per second, a lot more reads with caching and replicas.
- One Redis node: 100k+ simple ops per second.
- One Kafka partition: tens of MB/s. A cluster scales out with partitions.
- A single big cloud VM today can have hundreds of cores and terabytes of RAM. Vertical scaling goes further than it used to.

### Example estimate

Say you are building a photo sharing app.

- 50 million daily active users
- each user views 50 photos a day and uploads 1 photo every 10 days
- average photo after compression is 500 KB, plus a few thumbnails, call it 1 MB total

Reads: 50M x 50 = 2.5B views per day, about 30k per second average, maybe 100k at peak. That is CDN territory, not origin server territory.

Writes: 50M / 10 = 5M uploads per day, about 60 per second. Easy for the metadata database.

Storage: 5M x 1 MB = 5 TB per day, about 1.8 PB per year. Object storage, with lifecycle rules to move old stuff to cheaper tiers.

From 5 minutes of math you already know the shape of the system. Read path is all about CDN and caching. Write path is small. Storage cost will be a big line item.

---

## 4. Core Building Blocks

### DNS

Turns names into IPs. Also used for coarse traffic steering (geo DNS, latency based routing, failover). TTLs matter. Low TTL means faster failover but more DNS traffic. Clients and resolvers don't always respect TTL, so DNS failover is never instant.

### CDN and edge

A CDN caches content close to users. Use it for static assets, images, video, and more and more for dynamic content too.

- Cache key design matters. Bad cache keys kill hit rate.
- Use cache headers on purpose (Cache-Control, ETag, stale-while-revalidate).
- Invalidation is the hard part. Prefer versioned URLs (`app.4f9a1c.js`) over purging.
- Modern CDNs can run code at the edge (Cloudflare Workers, Fastly Compute, Lambda@Edge, Vercel/Netlify edge functions). Good for auth checks, redirects, A/B routing, personalization. Not great for anything that needs to talk to a far away database on every request.

### Load balancers

Spread traffic across servers.

- **Layer 4** (TCP/UDP). Fast, doesn't look inside the request.
- **Layer 7** (HTTP). Can route by path, header, cookie. Can terminate TLS, do retries, rate limit.

Common algorithms: round robin, least connections, weighted, consistent hashing (for sticky routing to a cache or shard).

Health checks matter more than the algorithm. A load balancer that keeps sending traffic to a dead box is worse than no load balancer.

### API gateway

A layer 7 front door for APIs. Handles auth, rate limiting, request routing, request/response transforms, sometimes billing. Watch out for putting business logic in it. It turns into a mess fast.

### Service mesh

Sidecar or sidecarless proxies (Istio, Linkerd, Cilium) that handle service to service traffic. You get mTLS, retries, timeouts, traffic splitting and telemetry without changing app code. The cost is operational complexity. Small teams often don't need it. Istio's ambient mode and eBPF based options have cut down the sidecar overhead a lot, so it's worth a fresh look if you wrote it off a few years ago.

### Caches

Caching is the cheapest way to make reads fast. It's also the easiest way to serve wrong data.

**Where to cache**

- client (browser, mobile app)
- CDN / edge
- in process memory on the app server
- distributed cache (Redis, Valkey, Memcached)
- database buffer pool (you get this for free)

**Patterns**

- **Cache aside (lazy loading).** App checks cache, on miss reads DB and fills cache. Most common. Simple.
- **Read through.** Cache itself loads from DB on a miss.
- **Write through.** Write goes to cache and DB together. Cache is always fresh, writes are slower.
- **Write behind (write back).** Write goes to cache, flushed to DB later. Fast writes, risk of loss.
- **Refresh ahead.** Reload hot keys before they expire.

**Problems you need to plan for**

- **Stale data.** Pick a TTL you can live with. Invalidate on write if you need fresher data.
- **Thundering herd / cache stampede.** A hot key expires and 10,000 requests hit the DB at once. Fix with request coalescing (single flight), locks, or early random refresh.
- **Hot keys.** One key gets most of the traffic. Fix with local caching in front, key replication, or splitting the key.
- **Cold start.** A new cache node or a flushed cache means a sudden DB load spike. Warm it up or ramp traffic slowly.
- **Cache penetration.** Lots of requests for keys that don't exist. Cache the "not found" result too, or use a bloom filter.

**Eviction.** LRU is the default. LFU is better for some skewed workloads. Know that you can't cache everything, so size for the working set.

Note on Redis: after the 2024 license change a lot of companies moved to Valkey (the Linux Foundation fork). Redis later added AGPL as a license option again in 2025. Both are widely used. Know that the choice is partly a licensing and vendor question now, not just technical.

### Object storage

S3, GCS, Azure Blob and the S3 compatible options (R2, MinIO, etc). Cheap, durable, basically infinite. Higher latency than block storage. The default home for files, backups, logs, data lake tables, ML datasets and model weights.

Things to know:

- S3 has been strongly consistent for reads after writes since late 2020. Older docs that say "eventually consistent" are out of date.
- Conditional writes (if-none-match, if-match) are now supported on S3, which makes it usable for some coordination tasks that used to need a database.
- Use presigned URLs so clients upload and download directly, not through your servers.
- Use lifecycle rules for tiering and deletion. Storage cost adds up.

### Search

For full text search, faceting and relevance ranking, you want a search engine (Elasticsearch, OpenSearch, Solr, or newer ones like Typesense, Meilisearch, Vespa). Postgres full text search is good enough for a lot of apps and saves you from running another system.

Most search setups are a secondary index. The source of truth lives in your main database and gets synced over (usually via CDC or an event stream). So plan for the index being a little behind.

---

## 5. Data Storage

### Picking a database

Start from access patterns, not from what is popular.

Questions to ask:

- What are the main queries? Point lookups, range scans, joins, aggregations, full text, similarity search?
- How much data now, and in 3 years?
- Read/write ratio?
- Do you need multi row transactions?
- What consistency do you need?
- Who is going to run it at 3am?

### Relational (Postgres, MySQL, and distributed SQL)

The default choice for most new systems. Has been for a while and still is.

Why:

- transactions, constraints, joins
- mature tooling, everyone knows SQL
- scales a lot further than people assume with good indexes, read replicas and partitioning
- Postgres especially has a huge extension ecosystem (pgvector for embeddings, PostGIS for geo, TimescaleDB for time series, Citus for sharding)

When a single primary is not enough, you have options:

- read replicas for read scaling
- table partitioning inside one database
- application level sharding
- Vitess (MySQL) or Citus (Postgres) for managed sharding
- distributed SQL like CockroachDB, YugabyteDB, Google Spanner, TiDB, or AWS Aurora DSQL. These give you SQL and transactions across nodes and regions, at the cost of higher write latency and a more complex system.

### Key value and wide column

DynamoDB, Cassandra, ScyllaDB, Bigtable, HBase.

Good for:

- huge scale with simple access patterns
- predictable low latency point reads and writes
- write heavy workloads (time series, event logs, IoT)

The catch: you design tables around your queries up front. Adding a new query pattern later can mean a new table or a full backfill. Joins don't really exist.

### Document stores

MongoDB, Couchbase, Firestore. Flexible schema, nested documents. Good when your data is naturally shaped like documents and you mostly read and write whole documents. MongoDB has had multi document transactions for years now, so the old "no transactions" argument is gone. Schema still matters, you just enforce it in code or with schema validation.

### Time series

TimescaleDB, InfluxDB, ClickHouse, Prometheus (for metrics). Optimized for append heavy, time ordered data with downsampling and retention.

### Graph

Neo4j, Neptune, and others. Good when the queries are really about relationships many hops deep (fraud rings, recommendations, access graphs). For 1 or 2 hops, a relational DB is usually fine.

### Columnar / OLAP

ClickHouse, Snowflake, BigQuery, Redshift, DuckDB, Druid, Pinot. Built for scanning and aggregating huge amounts of data. Not for high rate single row updates. See section 13.

### Vector databases

Store embeddings and do approximate nearest neighbor (ANN) search. Pinecone, Weaviate, Milvus, Qdrant, plus vector support in Postgres (pgvector), Elasticsearch/OpenSearch, MongoDB, Redis, and most cloud databases. See section 14.

Honest take: for most teams starting out, pgvector or vector search in the database you already run is the right call. Move to a dedicated vector store when you have a real reason (very large scale, very high QPS, specialized filtering needs).

### Indexes

Indexes are the main tool for making reads fast. They cost you on writes and storage.

- **B-tree.** Default in relational DBs. Good for equality and range queries.
- **Hash.** Equality only.
- **LSM tree.** Used by Cassandra, RocksDB, ScyllaDB and others. Fast writes, reads need to check multiple levels, compaction runs in the background.
- **Inverted index.** Full text search.
- **GIN / GiST.** Postgres indexes for JSON, arrays, full text, geo.
- **HNSW / IVF.** ANN indexes for vectors.

Things that come up a lot:

- composite index column order matters (leftmost prefix rule)
- covering indexes avoid going back to the table
- too many indexes slow down writes and confuse the planner
- always look at the actual query plan (`EXPLAIN ANALYZE`) instead of guessing

### B-tree vs LSM in one paragraph

B-trees update in place. Reads are fast and predictable, writes do random IO. LSM trees buffer writes in memory and flush sorted files, then merge them later. Writes are fast and sequential, reads may touch several files, and compaction can cause latency spikes if you don't tune it. Write heavy at huge scale leans LSM. Mixed workloads with lots of range reads lean B-tree.

---

## 6. Replication, Partitioning and Consistency

### Replication

Copies of data on multiple nodes. For durability, availability, and read scaling.

**Single leader.** One primary takes writes, replicas follow. Simple. Failover is the hard part. Replicas can lag, so reading from them can show stale data.

**Multi leader.** Multiple nodes take writes, usually one per region. Good for multi region writes. You now have write conflicts and need a strategy (last write wins, merge, CRDTs, app level resolution).

**Leaderless.** Any node takes writes (Dynamo style, Cassandra). Uses quorums. With N replicas, W write acks and R read acks, if W + R > N you get overlapping reads and writes. That is not the same as linearizability, but it's close enough for a lot of use cases.

**Synchronous vs asynchronous.** Sync replication means no data loss on failover but slower writes and lower availability. Async means fast writes but you can lose the last few writes if the leader dies. Many systems do semi sync (wait for at least one replica).

### Read your writes and other gotchas

If a user writes to the primary then reads from a lagging replica, they don't see their own change. That looks like a bug to them. Fixes:

- read from the primary for a short time after a write
- track the replication position and only read from replicas that have caught up
- route that user's reads to the primary

Also watch out for **monotonic reads** (user sees new data, then refreshes and sees older data because they hit a different replica). Sticky routing per user helps.

### Partitioning (sharding)

Split data across nodes so no single node has to hold or serve everything.

**By key range.** Easy range queries. Risk of hot spots (all new writes going to the latest time range).

**By hash of key.** Spreads load evenly. Range queries across keys get expensive.

**Consistent hashing.** Nodes and keys go on a ring. Adding or removing a node only moves a small part of the data. Virtual nodes smooth out the distribution. Used by Dynamo, Cassandra, many caches.

**By tenant or entity.** All data for one customer lives together. Great for multi tenant SaaS. Watch out for huge tenants.

Things to plan for:

- **Hot partitions.** One celebrity user, one big customer, one viral post. You may need to split hot keys or add a caching layer in front.
- **Rebalancing.** Moving data while serving traffic is hard. Pick a scheme that makes it rare and cheap.
- **Cross partition queries.** Scatter gather is slow and gets slower as you add shards. Design so the common queries hit one partition.
- **Secondary indexes.** Either local per partition (writes easy, reads scatter) or global (reads easy, writes touch multiple partitions).

### CAP and PACELC

**CAP.** When there is a network partition, you have to pick between consistency and availability. That's it. It's not "pick 2 of 3", because you don't get to opt out of partitions.

**PACELC** is more useful day to day. If there's a Partition, choose Availability or Consistency. Else (normal operation), choose Latency or Consistency. Most of the time there is no partition, and the real trade-off you live with is latency vs consistency.

### Consistency models (strongest to weakest, roughly)

- **Linearizable.** Every read sees the latest write. Behaves like a single copy. Expensive across regions.
- **Sequential.** Everyone sees operations in the same order, but not necessarily real time order.
- **Causal.** If A caused B, everyone sees A before B. Unrelated things can show up in any order. A really useful middle ground.
- **Read your writes, monotonic reads.** Session guarantees. Often all a user actually needs.
- **Eventual.** Replicas converge if writes stop. No promise about when.

Be specific in design discussions. "Eventually consistent" alone doesn't say much. Say how stale, for which reads, and what the user sees.

### Consensus

When nodes have to agree on something (who is leader, what the committed log is), you need a consensus algorithm. **Raft** is the one to know. Paxos is the older one. Used in etcd, Consul, ZooKeeper (ZAB), CockroachDB, TiKV, Kafka (KRaft, which replaced ZooKeeper and is the only mode in Kafka 4.0+).

What you should be able to explain:

- a leader is elected by majority
- writes go to the leader and are committed when a majority acks
- a cluster of 2f+1 nodes tolerates f failures (3 nodes tolerate 1, 5 tolerate 2)
- you rarely implement this yourself, you use a system that already does

### Time and ordering

Clocks on different machines drift. Don't use wall clock time to order events across machines if correctness depends on it.

- **Lamport clocks.** Give you a consistent order but no real time meaning.
- **Vector clocks.** Can tell if two events are concurrent.
- **Hybrid logical clocks (HLC).** Mix of physical and logical time. Used in CockroachDB and others.
- **TrueTime.** Google Spanner's bounded uncertainty clock using GPS and atomic clocks. Lets it do external consistency by waiting out the uncertainty.

### Leader election and leases

Leases give one node the right to act as leader for a period of time. If the leader pauses (GC, VM freeze) and the lease expires, another node can take over while the old one still thinks it's leader. Use **fencing tokens** (a number that goes up every time the lease changes hands) so storage can reject writes from the old leader.

### CRDTs

Conflict free replicated data types. Data structures that can be updated on different replicas and always merge to the same result without coordination. Counters, sets, registers, sequences (for text editing). Used in collaborative editors and local first apps (Automerge, Yjs). Good to know, especially for offline and multi region writes.

---

## 7. Messaging, Queues and Streams

### Why use them

- decouple producers and consumers
- absorb traffic spikes
- do slow work in the background
- fan out one event to many consumers
- replay history (streams only)

### Queues vs logs

**Message queue** (SQS, RabbitMQ, Google Pub/Sub, Azure Service Bus). Messages are consumed and then gone. Good for task distribution. Competing consumers pull work.

**Log / stream** (Kafka, Redpanda, Kinesis, Pulsar, NATS JetStream). Messages are appended to a durable ordered log and kept for a retention period. Consumers track their own offset. Many independent consumers can read the same data. You can replay.

Rule of thumb: if you need "do this job once", use a queue. If you need "this happened, anyone who cares can react, and we may want to replay it", use a log.

### Delivery guarantees

- **At most once.** Might lose messages, never duplicates.
- **At least once.** Never lose, might get duplicates. The common default.
- **Exactly once.** True end to end exactly once doesn't really exist across system boundaries. What you can get is "effectively once" by combining at least once delivery with idempotent processing. Kafka's transactions give exactly once within Kafka (read, process, write back to Kafka), not to your external database.

So: **make consumers idempotent.** Use an idempotency key or dedupe table. Use upserts. Make "apply this twice" have the same effect as "apply this once".

### Ordering

- Kafka guarantees order within a partition, not across partitions. Pick the partition key so things that need ordering share a key (e.g., all events for one order ID).
- SQS standard queues don't guarantee order. FIFO queues do, within a message group, at lower throughput.
- If you retry a failed message by putting it at the back of the queue, you've broken ordering. Decide if you care.

### Backpressure and dead letters

- If consumers can't keep up, lag grows. Monitor consumer lag as a top level metric.
- Messages that fail over and over should go to a **dead letter queue** after N tries so they don't block everything. Have a process for looking at the DLQ. A DLQ nobody checks is just a slow way to lose data.

### Pub/sub fan out

One event, many subscribers. For example "order placed" goes to billing, email, analytics and inventory. Each subscriber gets its own queue or consumer group so a slow one doesn't hold up the others.

---

## 8. Distributed Transactions and Workflows

### The problem

You need to update two things (two services, a database and a queue, two databases) and either both happen or neither. Across a network that's hard.

### Two phase commit (2PC)

A coordinator asks everyone to prepare, then tells everyone to commit. Works but blocks if the coordinator dies at the wrong time, and it couples services tightly. Used inside some databases. Usually avoided between microservices.

### Sagas

Break the big transaction into a series of local transactions. If one step fails, run compensating actions to undo the earlier steps.

- **Choreography.** Each service listens for events and reacts. No central brain. Hard to follow once there are more than a few steps.
- **Orchestration.** A central orchestrator tells each service what to do next. Easier to understand and debug.

Compensations aren't always a clean undo. You can't un-send an email. Design for "semantic undo" (send a correction email, issue a refund).

### Transactional outbox

The classic problem: you update your DB and then publish an event. If the publish fails, the DB says one thing and the rest of the world doesn't know.

The fix: write the event to an **outbox table** in the same DB transaction as your data change. A separate process (or CDC tool like Debezium) reads the outbox and publishes. Now the event is published if and only if the data change committed. Consumers still need to be idempotent because publishing is at least once.

### Change data capture (CDC)

Read the database's own change log (Postgres WAL, MySQL binlog) and turn it into a stream of events. Debezium is the common open source tool, most cloud providers have managed options too. Great for keeping caches, search indexes and analytics in sync without dual writes from the app.

### Durable execution

This one has gone mainstream in the last few years. Tools like **Temporal**, **Restate**, **AWS Step Functions**, **Azure Durable Functions**, and **DBOS** let you write long running workflows as normal looking code. The engine persists each step so the workflow survives crashes, restarts and deploys, and picks up where it left off.

Why it matters:

- sagas, retries, timers and human approval steps get way simpler
- you stop hand building state machines on top of queues and cron
- it's also showing up a lot as the backbone for AI agent workflows, since those are long running, have many steps and fail in weird ways

The trade-off is a new piece of infrastructure and some rules about how you write code (workflow code has to be deterministic).

### Idempotency

Worth its own section because it shows up everywhere.

- Clients send an **idempotency key** with a write. The server stores the key and the result. If the same key comes again, return the stored result instead of doing the work again.
- Stripe's API is the well known example.
- Store keys with a TTL. Scope them per user or per account.
- Make sure the "check key" and "do the work" are atomic, or two concurrent retries can both get through.

---

## 9. Reliability and Resilience

### Know your targets

- **SLI** (service level indicator): what you measure. E.g. percent of requests that succeed in under 300ms.
- **SLO** (objective): the target. E.g. 99.9% over 30 days.
- **SLA** (agreement): the contract with customers, with penalties. Should be looser than your SLO.
- **Error budget:** 100% minus the SLO. 99.9% over 30 days is about 43 minutes of "bad". When you burn through it, slow down on risky changes.

Rough downtime per year:

| Availability | Downtime per year |
|---|---|
| 99% | ~3.65 days |
| 99.9% | ~8.8 hours |
| 99.95% | ~4.4 hours |
| 99.99% | ~53 minutes |
| 99.999% | ~5 minutes |

Remember your availability is limited by your dependencies. If you call 5 services in series that are each 99.9%, you're at about 99.5% at best.

### Timeouts

Every network call needs a timeout. No exceptions. Default timeouts in a lot of client libraries are way too long or infinite.

- Set timeouts based on the downstream's real latency (a bit above its p99).
- Use **deadline propagation**. If the user request has 2 seconds left, don't start a downstream call with a 5 second timeout. gRPC does this natively.

### Retries

Retries fix transient errors. They also cause outages when done wrong.

- Only retry things that are safe to retry (idempotent or with an idempotency key).
- Use **exponential backoff with jitter**. Without jitter, every client retries at the same moment.
- Cap the number of retries.
- Retry at one layer, not every layer. If 4 layers each retry 3 times, one failure turns into 81 calls.
- Use a **retry budget** (e.g. retries can't be more than 10% of total requests).

### Circuit breakers

If a dependency is failing, stop calling it for a while. Fail fast or return a fallback. After a cooldown, let a few test requests through (half open). If they work, close the circuit.

### Bulkheads

Isolate resources so one bad thing can't take everything down. Separate thread pools or connection pools per dependency. Separate clusters for different customer tiers. Cell based architecture (below) is bulkheads at the infrastructure level.

### Rate limiting

Protects you from abuse and from your own clients having a bad day.

Algorithms:

- **Token bucket.** Tokens refill at a steady rate, each request takes one. Allows bursts up to bucket size. The most common choice.
- **Leaky bucket.** Requests drain at a fixed rate. Smooths traffic.
- **Fixed window counter.** Simple, but allows double bursts at window edges.
- **Sliding window log / sliding window counter.** More accurate, a bit more expensive.

Where: at the edge or gateway for global limits, in the service for per feature limits. For distributed limiting, use Redis with atomic scripts or a dedicated rate limit service. Return 429 with a `Retry-After` header.

### Load shedding and backpressure

When you're overloaded, it's better to reject some requests fast than to slow down all of them.

- Drop low priority traffic first (background jobs, prefetch, analytics).
- Use queue length or latency as the signal, not just CPU.
- Use adaptive concurrency limits that adjust based on latency.
- Propagate backpressure upstream instead of letting queues grow forever. Unbounded queues just hide the problem until memory runs out.

### Graceful degradation

Decide ahead of time what you can turn off. If recommendations are down, show popular items. If the personalization service is slow, show a generic page. Feature flags make this easy to do in an incident.

### Deployments

Most outages are caused by changes. Make changes safe.

- small, frequent deploys
- canary releases and progressive rollouts (1%, 10%, 50%, 100%)
- automatic rollback on SLO regression
- feature flags to decouple deploy from release
- config changes deserve the same care as code changes (a lot of big outages were config pushes)
- database migrations done in backwards compatible steps (expand, migrate, contract)

### Multi AZ and multi region

- **Multi AZ** is the baseline for anything important. Most managed services do this for you.
- **Multi region** is expensive and hard. Do it when you have a real reason (regulation, latency for global users, surviving a full region outage).

Multi region options:

- **Active passive.** One region serves traffic, the other is on standby. Simpler. Failover takes time and is often not tested enough.
- **Active active.** All regions serve traffic. Needs a plan for data (partition users by region, or use a multi region database, or accept conflicts).

### Cell based architecture

Split the system into many independent copies (cells), each serving a slice of customers. A bad deploy or a poison request only hurts one cell. AWS talks about this a lot and it's become a standard pattern for large SaaS. You need a thin routing layer that maps customers to cells. Pair it with **shuffle sharding** so that no two customers share the exact same set of resources, which limits blast radius even more.

### Disaster recovery

- **RPO** (recovery point objective): how much data you can lose. Minutes? Zero?
- **RTO** (recovery time objective): how long you can be down.

Backups you've never restored are not backups. Test restores on a schedule. Same for failover. If you've never failed over in production, assume it doesn't work.

### Chaos and game days

Break things on purpose in a controlled way. Kill nodes, inject latency, cut off a dependency. Run game days where the team walks through a scenario. You'll find gaps in alerts and runbooks before a real incident finds them for you.

---

## 10. API Design

### Styles

- **REST over HTTP/JSON.** Default for public APIs. Everybody understands it. Easy to cache.
- **gRPC.** Binary, fast, strongly typed with protobuf. Streaming built in. Great for service to service. Harder from browsers (needs gRPC-Web or Connect).
- **GraphQL.** Client picks what fields it wants. Good for complex UIs with many data needs. Watch out for expensive queries, N+1 problems on the server, and caching being harder. Use query cost limits and persisted queries.
- **Async / event APIs.** Webhooks, pub/sub, WebSockets, Server Sent Events. Use AsyncAPI to document them.

### Good habits

- Use nouns for resources and HTTP verbs for actions.
- Version from day one (`/v1/`, or a header). Never break existing clients without warning.
- Use **cursor based pagination** for large or changing lists. Offset pagination gets slow and skips/duplicates items when data changes.
- Support **idempotency keys** on POSTs that create things or move money.
- Return clear errors with a machine readable code and a human message. RFC 9457 (Problem Details) is a decent standard.
- Use proper status codes. 4xx means the client did something wrong, 5xx means you did.
- Put rate limit info in response headers.
- Design for partial failure in batch endpoints (return per item results).

### Real time

- **Polling.** Simple, wasteful.
- **Long polling.** Server holds the request until there's data.
- **Server Sent Events (SSE).** One way server to client stream over HTTP. Simple and works great for things like notifications and streaming LLM responses.
- **WebSockets.** Two way. Good for chat, games, collaboration. You need to manage connection state, which makes scaling and deploys harder (sticky routing, connection draining).
- **WebTransport** (over HTTP/3) is the newer option. Support is growing but WebSockets is still the safe default.

### Webhooks

When you send them:

- sign the payload (HMAC) so receivers can verify it
- retry with backoff, and let customers see delivery history
- include an event ID so receivers can dedupe
- don't put sensitive data in the payload if you can avoid it, send an ID and let them fetch

When you receive them:

- verify the signature
- respond 2xx fast and process async
- be idempotent

### Protocols worth knowing

- **HTTP/2.** Multiplexing over one connection. Standard now.
- **HTTP/3 (QUIC).** Runs over UDP. Better on lossy networks and mobile, faster connection setup. Widely supported by CDNs and browsers.
- **TLS 1.3.** Faster handshake. Should be your minimum for anything new.

---

## 11. Observability

You can't run what you can't see.

### The three signals (plus one)

- **Metrics.** Numbers over time. Cheap to store, good for dashboards and alerts.
- **Logs.** Detailed records of events. Structured (JSON) logs, not free text.
- **Traces.** Follow one request across services. Essential for microservices.
- **Profiles.** Continuous profiling (CPU, memory) is now often called the fourth signal. Tools like Pyroscope, Parca, and most APM vendors support it.

### OpenTelemetry

OpenTelemetry (OTel) is the standard now. Use it for instrumentation so you aren't locked to one vendor. You collect with the OTel SDK and Collector, then send to whatever backend you want (Grafana stack, Datadog, Honeycomb, New Relic, Elastic, cloud native tools, etc).

### What to measure

For services, the **RED** method:

- **R**ate (requests per second)
- **E**rrors (failed requests)
- **D**uration (latency distribution)

For resources, the **USE** method:

- **U**tilization
- **S**aturation
- **E**rrors

Google's **four golden signals** are basically the same idea: latency, traffic, errors, saturation.

### Latency

- Look at percentiles (p50, p95, p99, p99.9), not averages. Averages hide the pain.
- Tail latency gets worse with fan out. If a request calls 100 backends and each has a 1% chance of being slow, most requests will be slow.
- Use hedged requests (send a backup request after a short delay) for latency sensitive fan out, but only for idempotent reads.

### Alerting

- Alert on symptoms users feel (SLO burn rate), not on every cause (CPU at 80%).
- Use multi window burn rate alerts. Fast burn pages someone. Slow burn opens a ticket.
- Every page should be actionable. If nobody needs to do anything, it shouldn't page.
- Keep runbooks linked from alerts.

### Cardinality and cost

High cardinality labels (user ID, request ID) in metrics blow up cost fast. Put those in traces and logs, not metric labels. Observability bills are a real line item now, so sample traces, set log retention, and drop noisy logs.

---

## 12. Security

At senior level you're expected to bring this up yourself, not wait for someone to ask.

### Authentication

- **OAuth 2.0** for delegated access. **OpenID Connect (OIDC)** on top of it for login. OAuth 2.1 consolidates the good practices (PKCE everywhere, no implicit flow).
- **Passkeys (WebAuthn/FIDO2)** are now widely supported across platforms and are the direction for consumer login. Phishing resistant, no passwords to steal.
- **Sessions vs tokens.** Server side sessions are easy to revoke. JWTs are stateless but hard to revoke, so keep them short lived and use refresh tokens.
- For service to service, use **mTLS** or workload identity (SPIFFE/SPIRE, cloud IAM roles), not long lived shared secrets.

### Authorization

- **RBAC** (role based). Simple, works for most apps.
- **ABAC** (attribute based). Policies based on user, resource and context attributes.
- **ReBAC** (relationship based). Permissions come from relationships ("user is editor of folder that contains doc"). Google's **Zanzibar** paper started this. Open source and hosted options include OpenFGA, SpiceDB, Permify, and others.
- Policy engines like **OPA** and **Cedar** let you write policies outside app code.

Check authorization on the server for every request. Broken object level authorization (user A can read user B's stuff by changing an ID) is still one of the most common API bugs.

### Data protection

- TLS everywhere, including inside your network.
- Encrypt at rest (usually a checkbox on managed services). Use a KMS. Consider envelope encryption for sensitive fields.
- Keep secrets in a secrets manager (Vault, cloud secret managers), never in code or plain env files in git.
- Know where PII lives. Minimize what you collect. Plan for deletion requests (GDPR, CCPA and similar laws).
- Data residency rules (EU and others) can force your architecture into regional deployments.

### Zero trust

Don't trust something just because it's inside your network. Authenticate and authorize every call. Identity aware proxies for internal tools instead of VPN only access.

### Multi tenancy

- **Silo:** separate infra per tenant. Strong isolation, high cost.
- **Pool:** shared infra, tenant ID on every row. Cheap, needs careful isolation in code. Row level security in Postgres helps.
- **Bridge / hybrid:** shared for most, dedicated for big or regulated tenants.

Watch out for noisy neighbors. Per tenant rate limits and quotas.

### Supply chain

This has become a big deal. Know the basics:

- pin dependencies and scan them
- generate an **SBOM** (software bill of materials)
- sign build artifacts (Sigstore/cosign)
- follow **SLSA** levels for build integrity
- least privilege for CI/CD credentials, prefer short lived OIDC tokens to long lived keys

### Common threats to design for

- injection (SQL, command, and now prompt injection for LLM systems)
- SSRF (server side request forgery), especially in anything that fetches URLs for users
- credential stuffing and account takeover
- DDoS (use your CDN/WAF, rate limits)
- insecure direct object references
- leaking secrets in logs

---

## 13. Data Platforms and Analytics

### OLTP vs OLAP

- **OLTP:** many small reads and writes, low latency, current state. Your app database.
- **OLAP:** big scans and aggregations over lots of history. Your warehouse or lakehouse.

Don't run heavy analytics on your production OLTP database. Copy the data out.

### Batch vs stream

- **Batch:** process data in chunks on a schedule. Simple, cheap, high latency. Spark, dbt, SQL in the warehouse.
- **Stream:** process events as they arrive. Low latency, more complex. Flink, Kafka Streams, Spark Structured Streaming, and newer streaming databases like RisingWave and Materialize.

Lambda architecture (batch + stream side by side) mostly lost out to **Kappa** style (just stream, replay for reprocessing) and to lakehouse setups where both batch and streaming write to the same tables.

### Lakehouse and open table formats

The big shift in the last few years. Store data in open file formats (Parquet) on object storage, with a table format on top that gives you transactions, schema evolution and time travel.

- **Apache Iceberg** has become the front runner, with broad support from Snowflake, Databricks, AWS (S3 Tables), Google, and most query engines.
- **Delta Lake** is still big, mainly in the Databricks world.
- **Apache Hudi** is around too.

Why it matters for system design: you can have one copy of data and query it with many engines (Spark, Trino, Snowflake, DuckDB, etc) instead of copying it into each one.

### ETL vs ELT

Old way: transform before loading into the warehouse (ETL). Now: load raw data first, transform inside the warehouse with SQL (ELT), usually with dbt or similar. Cheaper compute and storage made this the default.

### Real time analytics

For user facing dashboards and analytics with fresh data and fast queries: ClickHouse, Apache Pinot, Apache Druid, StarRocks. These can handle high QPS on fresh data in ways a classic warehouse can't.

### DuckDB

Worth a mention on its own. An in process analytical database, like SQLite for analytics. Shockingly fast on a single machine. A lot of "we need a big data cluster" problems turn out to fit on one laptop or one VM with DuckDB.

---

## 14. What Is Current in 2026

This is the section to read if you've been heads down for a couple of years.

### AI and LLM systems are now a normal design topic

Expect to design or review systems that call LLMs. A lot of the classic ideas still apply, there are just new constraints.

**What makes LLM calls different from normal API calls**

- slow (seconds, not milliseconds) and the latency varies a lot with output length
- expensive per call, priced by tokens
- non deterministic, the same input can give different outputs
- they can be wrong in ways that look right
- rate limits are usually in tokens per minute, not just requests per minute
- providers have outages and capacity limits like anyone else

**Patterns to know**

- **Streaming responses** (SSE) so users see output right away. Time to first token is the latency metric users feel.
- **Model routing.** Send simple requests to small cheap models and hard ones to big models. Have a fallback provider or model when the main one is down or rate limited.
- **Prompt caching.** Major providers cache repeated prompt prefixes (long system prompts, documents) for big cost and latency savings. Design prompts so the stable part comes first.
- **Response caching.** Exact match caching works for repeated questions. Semantic caching (match by embedding similarity) is possible but risky, you can return the wrong answer to a slightly different question.
- **Batch APIs.** For offline work, provider batch endpoints are much cheaper than real time calls.
- **Token budgets and quotas** per user and per tenant, same as any other expensive resource.
- **Async by default** for long jobs. Return a job ID, process in the background, notify when done.

**Retrieval Augmented Generation (RAG)**

Give the model relevant context from your own data at request time.

The basic pipeline:

1. **Ingest:** pull documents, split into chunks, create embeddings, store in a vector index with metadata.
2. **Retrieve:** embed the user query, find the nearest chunks, usually combined with keyword search (**hybrid search**) for better results.
3. **Rerank:** use a reranker model to order the top results better.
4. **Generate:** put the best chunks in the prompt and call the model.

Design questions that come up:

- How do you keep the index fresh when source data changes? (CDC into the embedding pipeline)
- How do you enforce permissions so users only retrieve docs they're allowed to see? (filter by ACL at query time, this is a very common miss)
- How do you handle re-embedding when you switch embedding models? (version your embeddings, rebuild in the background)
- How big should chunks be, and how do you keep context like headings with them?

As models got longer context windows, some teams just put more in the prompt instead of building complex retrieval. That works for smaller data sets but costs more per call. RAG still wins for big or fast changing data and for access control.

**Agents and tool use**

Agents are LLMs that call tools (APIs, code, search) in a loop to complete a task. From a system design point of view:

- they're long running, multi step workflows (durable execution fits well here)
- each tool call is a side effect that needs the same care as any API call: auth, idempotency, rate limits, audit logs
- **Model Context Protocol (MCP)** has become the common standard for connecting models and agents to tools and data sources. Know what it is.
- sandbox anything that runs model generated code
- put humans in the loop for risky actions (payments, deletes, sending emails)
- limit what an agent can do with scoped credentials, never give it a god token

**Security for LLM systems**

- **Prompt injection** is the big one. Any text the model reads (web pages, emails, documents, tool outputs) can contain instructions. Treat model output as untrusted. Don't let it trigger sensitive actions without checks.
- Watch for data leaking between users through shared context or caches.
- Log prompts and responses for debugging and audit, but treat those logs as sensitive data.
- OWASP has a Top 10 for LLM applications. Worth a read.

**Evaluation and observability**

- You need **evals**: a set of test inputs with expected behavior, run on every prompt or model change. This is the LLM version of a test suite.
- Trace each request: prompt, retrieved context, tool calls, tokens, latency, cost. OpenTelemetry has semantic conventions for generative AI now, and there are LLM focused tools (Langfuse, Arize, LangSmith, Braintrust and others).
- Track cost per request and per feature. It can surprise you.

**Self hosting models**

If you run your own models:

- GPU capacity is the bottleneck and it's expensive. Plan capacity carefully.
- Use an inference server built for it (vLLM, SGLang, TensorRT-LLM, Hugging Face TGI) that does continuous batching and KV cache management.
- Quantization trades a little quality for big memory and cost savings.
- Autoscaling is slow because loading model weights takes time. Keep warm capacity.
- The trade-offs are cost at scale and data control vs a lot more operational work.

### Postgres for (almost) everything

The "just use Postgres" idea got a lot stronger. With extensions it handles relational, JSON, full text, vectors (pgvector), geo (PostGIS), time series (Timescale), queues (pgmq, SKIP LOCKED patterns), and more. Fewer moving parts means fewer things to break. Start there and move off only when you hit a real limit.

Serverless Postgres options (Neon, Supabase, Aurora Serverless, and others) with branching and scale to zero are popular for dev environments and spiky workloads.

### Simpler architectures are back in style

After years of "everything is a microservice", a lot of teams pulled back.

- **Modular monolith** first: one deployable, clear module boundaries inside it. Split out services only when you have a reason (different scaling needs, different team ownership, different release pace).
- Microservices are an organizational tool as much as a technical one. If you have one team, you probably don't need 30 services.
- Some high profile teams wrote publicly about moving back from serverless or microservices to simpler setups to cut cost. The lesson isn't "microservices bad". It's "pick based on your actual situation".

### Cost is a design requirement now

FinOps went mainstream. Expect to be asked "what does this cost to run?"

- egress and cross AZ traffic are hidden costs that add up fast
- observability, logging and GPU spend are often bigger than people expect
- reserved capacity / savings plans vs on demand vs spot
- ARM based instances (Graviton, Axion, Cobalt) are often cheaper for the same work
- right sizing and autoscaling
- storage tiering and retention policies
- know the unit economics: cost per request, per user, per tenant

### Kubernetes is the default platform, and platform engineering is a thing

Kubernetes is the baseline for running containers at most mid to large companies. The trend now is **platform engineering**: an internal platform team builds paved paths (templates, golden paths, internal developer portals like Backstage) so product teams don't have to be Kubernetes experts.

Also worth knowing:

- **Gateway API** is replacing Ingress as the standard way to configure traffic in Kubernetes.
- **eBPF** based networking and observability (Cilium and others) is widely used.
- **GitOps** (Argo CD, Flux) for deploying from git.

### Serverless and edge, with realistic expectations

- Serverless functions (Lambda, Cloud Run, Azure Functions, Cloudflare Workers) are great for spiky, event driven and glue workloads.
- Cold starts got a lot better (SnapStart, lighter runtimes) but still matter for some use cases.
- At steady high load, serverless often costs more than containers.
- Edge compute is good for latency sensitive logic that doesn't need a central database on every call. Edge databases and replicas (Cloudflare D1, Turso, and similar) try to close that gap.

### WebAssembly outside the browser

Wasm is being used for plugins, edge functions and sandboxing untrusted code. The component model and WASI have matured. Not something you need in every design, but good to know as an option for running user supplied code safely.

### Local first and sync engines

Apps that keep data on the device and sync in the background, so they work offline and feel instant. Built on CRDTs or sync engines (Automerge, Yjs, ElectricSQL, Zero, PowerSync, Replicache and others). Shows up in collaborative tools and mobile apps.

### Regulation keeps shaping architecture

- data residency and sovereignty requirements (EU and more countries) push toward regional deployments
- the EU AI Act puts requirements on some AI systems (transparency, risk management, logging)
- the EU Cyber Resilience Act adds security requirements for software products
- privacy laws keep spreading

You don't need to be a lawyer. You do need to ask early "are there any legal constraints on where data lives or how it's processed?"

### Post quantum crypto

Not urgent for most designs today, but it's moving. NIST published the first post quantum standards in 2024 (ML-KEM, ML-DSA). Browsers and big providers already use hybrid post quantum key exchange in TLS. The practical takeaway is **crypto agility**: don't hardcode algorithms in ways that make them painful to swap later.

---

## 15. Worked Examples

Short versions. The goal is to show the key decisions, not every box.

### URL shortener

**Requirements:** create short links, redirect fast, maybe analytics. Very read heavy (100:1 or more).

**Key decisions:**

- **ID generation.** Options: hash of the URL (collisions, need to handle), random base62 string (check for collision on insert), or a counter (from a ticket server, or ranges handed out to each app server) encoded in base62. 7 base62 characters give about 3.5 trillion IDs.
- **Storage.** Simple key value: short code → long URL. Any KV store or a single Postgres table with an index is fine for a long time.
- **Reads.** Cache hot links in Redis and at the CDN. Use 301 (permanent, browsers cache it, you lose analytics) or 302 (temporary, every click hits you, you keep analytics).
- **Analytics.** Don't write to the DB on every click synchronously. Push click events to a stream, aggregate async.
- **Abuse.** Rate limit creation, scan URLs for malware/phishing.

### Rate limiter

**Requirements:** limit requests per user/key per time window, low latency, works across many servers.

**Key decisions:**

- **Algorithm.** Token bucket for most cases. Sliding window counter for more accurate limits.
- **State.** Central Redis with atomic Lua scripts (or a Redis module) so increments and checks are one step.
- **Latency.** The limiter is in the hot path of every request. Keep it fast. Consider a local in memory limiter that syncs with the central one periodically, accepting a little inaccuracy.
- **Failure mode.** If Redis is down, do you fail open (allow all) or fail closed (block all)? Usually fail open for user facing, fail closed for expensive or dangerous operations.
- **Response.** 429 with `Retry-After` and rate limit headers.

### News feed / timeline

**Requirements:** users follow others, see a feed of recent posts. Very read heavy. Some users have millions of followers.

**Key decisions:**

- **Fan out on write (push).** When someone posts, write the post ID to each follower's feed cache. Reads are fast. Writes are expensive for users with lots of followers.
- **Fan out on read (pull).** When someone opens their feed, fetch recent posts from everyone they follow and merge. Writes are cheap. Reads are expensive.
- **Hybrid** (what most real systems do): push for normal users, pull for celebrities, merge at read time.
- **Storage.** Posts in a sharded DB or wide column store. Feed caches in Redis as lists of post IDs. Hydrate post details from a cache.
- **Ranking.** Most feeds aren't purely chronological anymore. Candidate generation (fetch a few hundred candidates) then a ranking model. Be ready to talk about that split.
- **Pagination.** Cursor based.

### Chat / messaging

**Requirements:** 1:1 and group chat, real time delivery, history, online presence, delivery/read receipts.

**Key decisions:**

- **Connections.** WebSockets from clients to a gateway tier. Keep a mapping of user → gateway server (in Redis or similar) so you know where to push.
- **Message flow.** Client sends message → chat service writes it to storage → publishes to the recipient's gateway(s) → pushed to devices. If the recipient is offline, send a push notification.
- **Ordering.** Order within a conversation. Use a per conversation sequence number assigned by the server, not client timestamps.
- **Storage.** Partition by conversation ID. Wide column stores (Cassandra/ScyllaDB) are common at large scale, Postgres works for a long time at smaller scale.
- **Delivery guarantees.** At least once to the device, client dedupes by message ID.
- **Large groups.** Fan out gets expensive. Treat big channels more like a feed.
- **Presence.** Heartbeats with a TTL. Don't broadcast every presence change to everyone, it explodes. Only to people who are looking.
- **End to end encryption** changes a lot (server can't read messages, search and moderation move to the client). Signal protocol is the reference. MLS (Messaging Layer Security) is the newer standard for group E2EE.

### Notification system

**Requirements:** send notifications over push, email, SMS, in app. Respect user preferences. Don't spam.

**Key decisions:**

- Producers send a "notify" event to a queue with the user, type and data.
- A notification service checks preferences, quiet hours, rate limits and dedupe, renders templates, then puts per channel jobs on separate queues.
- Channel workers call providers (APNs, FCM, email, SMS). Separate queues so a slow SMS provider doesn't block push.
- Retries with backoff, provider failover, DLQ.
- Idempotency key per notification so retries don't double send.
- Track delivery status and let users see in app history.
- Priority lanes. A password reset can't wait behind a marketing blast.

### Distributed job scheduler

**Requirements:** run jobs at specific times or on a schedule, at least once, survive node failures.

**Key decisions:**

- Store jobs in a database with next run time indexed.
- Workers poll for due jobs and claim them with a lease (e.g. `SELECT ... FOR UPDATE SKIP LOCKED` in Postgres, or a conditional update).
- If a worker dies, the lease expires and another worker picks the job up. Jobs must be idempotent.
- Partition by job ID or time bucket when one DB isn't enough.
- Or skip building it and use a durable execution engine or managed scheduler.

### LLM powered support assistant

**Requirements:** answer customer questions using the company's help docs and the customer's account data. Hand off to a human when needed.

**Key decisions:**

- **Ingestion:** help docs → chunk → embed → vector index (pgvector to start). Re-index on doc change via CDC or webhook.
- **Request flow:** authenticate user → retrieve relevant doc chunks (hybrid search + rerank) → fetch account data through scoped tool calls (never give the model raw DB access) → call the model with streaming → stream back to the user.
- **Guardrails:** input and output filters, PII handling, refuse out of scope requests, never let the model take account actions (refunds, cancellations) without a confirm step or human approval.
- **Cost and latency:** route simple questions to a smaller model, use prompt caching for the long system prompt, set per user token budgets.
- **Reliability:** fallback model or provider, timeouts, graceful "let me connect you to a person" when things fail.
- **Quality:** eval set from real past tickets, run on every prompt change, track resolution rate and escalation rate, sample conversations for human review.
- **Observability:** trace prompt, retrieved docs, tool calls, tokens, latency and cost per conversation.

---

## 16. Trade-off Cheat Sheet

| Decision | Option A | Option B | Lean A when | Lean B when |
|---|---|---|---|---|
| Scaling | Vertical | Horizontal | Simple, data fits, short term | Past one machine, need fault tolerance |
| Database | SQL | NoSQL | Relationships, transactions, unknown future queries | Huge scale, simple known access patterns |
| Consistency | Strong | Eventual | Money, inventory, auth | Feeds, counters, analytics |
| Replication | Sync | Async | No data loss allowed | Low write latency matters more |
| Communication | Sync (HTTP/gRPC) | Async (queue/stream) | Need immediate answer | Can be done later, need decoupling |
| Fan out | On write | On read | Read heavy, few followers per user | Write heavy, celebrity accounts |
| Caching | Cache aside | Write through | Read heavy, can handle a bit of staleness | Need cache always fresh |
| Architecture | Monolith | Microservices | Small team, early product | Many teams, different scaling needs |
| Compute | Containers | Serverless | Steady load, long running | Spiky, event driven, low ops |
| API | REST | gRPC | Public, browser clients | Internal, high throughput |
| IDs | Auto increment | UUID / ULID / Snowflake | Single DB, simple | Distributed, no coordination, hide counts |
| Multi region | Active passive | Active active | Simpler, OK with failover time | Need low latency globally, near zero downtime |
| Vector search | In existing DB | Dedicated vector DB | Starting out, moderate scale | Very large scale, specialized needs |
| LLM | Hosted API | Self hosted | Speed, best models, low volume | Data control, very high volume, cost |

A note on IDs since it comes up a lot: **UUIDv7** (time ordered, standardized in RFC 9562 in 2024) is now a good default for distributed IDs. It sorts by time so it's index friendly, unlike random UUIDv4.

---

## 17. Common Mistakes

Things that come up in interviews and in real design reviews over and over.

- Jumping into boxes before understanding the problem.
- Designing for Google scale when the problem is 100 requests per second.
- Not doing any math, so you can't justify any choice.
- Adding Kafka, Redis, Elasticsearch and 12 microservices because they sound impressive.
- Saying "we'll use eventual consistency" without saying what the user sees.
- Forgetting the write path and only designing reads (or the other way around).
- No timeouts, retries without backoff, retries at every layer.
- Assuming the network is reliable and clocks are in sync.
- Ignoring hot keys and celebrity users.
- Dual writes (write DB, then publish event) without an outbox.
- Non idempotent consumers on an at least once queue.
- No plan for schema migrations or backfills.
- No observability story. "How would you know it's broken?" gets a blank stare.
- Ignoring security and multi tenant isolation until someone asks.
- Ignoring cost.
- Not stating trade-offs. Every choice has a downside. If you can't name it, you don't understand the choice yet.
- Talking for 20 minutes without checking in. In interviews, keep the interviewer involved. In reviews, let people poke holes early.

---

## 18. How to Stay Current

### Books

- **Designing Data-Intensive Applications** by Martin Kleppmann (the 2nd edition, with Chris Riccomini, is the one to get). If you only read one book, read this one.
- **Site Reliability Engineering** and **The Site Reliability Workbook** (Google, free online).
- **Database Internals** by Alex Petrov.
- **Fundamentals of Software Architecture** by Mark Richards and Neal Ford.
- **Software Architecture: The Hard Parts** by Ford, Richards, Sadalage and Dehghani.
- **Building Microservices** (2nd edition) by Sam Newman.
- **System Design Interview** Vol 1 and 2 by Alex Xu. Interview focused, good for practice.
- **AI Engineering** by Chip Huyen. Good for the LLM systems side.

### Papers worth reading at least once

- Google File System, MapReduce, Bigtable
- Amazon Dynamo
- Spanner
- Raft ("In Search of an Understandable Consensus Algorithm")
- Kafka (the original LinkedIn paper)
- Zanzibar (Google's authorization system)
- "The Tail at Scale" (Dean and Barroso)
- "Time, Clocks, and the Ordering of Events in a Distributed System" (Lamport)

### Engineering blogs

Pick a handful and actually read them. Good ones: AWS Architecture Blog and the Amazon Builders' Library, Netflix Tech Blog, Cloudflare Blog, Uber Engineering, Stripe Engineering, Discord Engineering, Figma Engineering, Slack Engineering, Meta Engineering, Shopify Engineering, Dropbox Tech, LinkedIn Engineering.

### Newsletters and other sources

- The Pragmatic Engineer (Gergely Orosz)
- ByteByteGo (Alex Xu)
- High Scalability archives
- InfoQ and QCon talks
- Papers We Love
- Public postmortems (AWS, Google Cloud, Cloudflare, GitHub publish them). Reading how real outages happened teaches more than most books.

### Habits that actually help

- Read one postmortem a week. Ask "would my system survive this?"
- When you use a new tool, read how it works under the hood, not just the getting started guide.
- Redraw a system you work on from memory once in a while. You'll find the parts you don't understand.
- Do mock design sessions with a peer. Explaining out loud is a different skill from knowing.
- Review this doc every 6 months and update section 14. Things change fast, especially in AI infrastructure.

---

## 19. Final Checklist

Run through this before you call a design done.

**Requirements**
- [ ] Functional requirements are clear and scoped
- [ ] Non-functional requirements have numbers (latency, availability, durability, scale)
- [ ] Constraints are known (regions, regulations, budget, existing systems)

**Estimates**
- [ ] Rough QPS (average and peak)
- [ ] Storage now and in a few years
- [ ] Bandwidth
- [ ] Rough cost

**Design**
- [ ] API defined
- [ ] Data model defined
- [ ] Read path walked end to end
- [ ] Write path walked end to end
- [ ] Database choice justified by access patterns
- [ ] Caching strategy and invalidation plan
- [ ] Partitioning plan if needed, hot keys addressed
- [ ] Consistency model stated per feature

**Reliability**
- [ ] Single points of failure identified and handled
- [ ] Timeouts, retries with backoff and jitter, circuit breakers
- [ ] Idempotency for writes and consumers
- [ ] Rate limiting and load shedding
- [ ] Graceful degradation plan
- [ ] Multi AZ at least, multi region if justified
- [ ] Backup and restore tested, RPO and RTO defined
- [ ] Safe deploys and rollback

**Operations**
- [ ] SLOs defined
- [ ] Metrics, logs, traces in place
- [ ] Alerts on user facing symptoms
- [ ] Runbooks

**Security**
- [ ] Authn and authz on every call
- [ ] Data encrypted in transit and at rest
- [ ] Secrets managed properly
- [ ] PII and data residency handled
- [ ] Tenant isolation (if multi tenant)
- [ ] Abuse cases considered (and prompt injection if there's an LLM involved)

**Wrap up**
- [ ] Trade-offs stated out loud
- [ ] Weak spots named
- [ ] What changes at 10x scale
