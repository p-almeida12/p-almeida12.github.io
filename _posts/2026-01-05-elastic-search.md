---
layout: post
title: Elasticsearch - My homework
date: 2025-11-30 10:59:00-0400
description: Discovering Elasticsearch and researching how it works in production systems, what are common problems and how to fix them.
tags:
categories:
giscus_comments: false
related_posts: false
toc:
  beginning: true
---
<span style="margin-left: 10px;"></span>
Elasticsearch is often introduced as “a search engine” or “a document store.” Both descriptions are incomplete and, in 
production systems, misleading. Elasticsearch is a distributed, near-real-time search and analytics engine built on top 
of Apache Lucene, optimized for fast reads, flexible querying, and horizontal scalability.

## What Elasticsearch Is (And What It Is Not)

### Elasticsearch as a Distributed Lucene Engine

<span style="margin-left: 10px;"></span>
At its core, Elasticsearch is a coordinator over many Lucene indices. Each index is split into shards, and each shard 
is a self-contained Lucene index with its own:
- inverted indices
- Segment files
- Term dictionaries
- Doc values

<span style="margin-left: 10px;"></span>
When you index a document:
- The document is routed to a shard
- It is analyzed (tokenized, filtered, normalized)
- It is written to an in-memory buffer
- It becomes searchable after a refresh (near-real-time, not immediate)
- It is eventually flushed to disk as immutable segments

<span style="margin-left: 10px;"></span>
This design has consequences:
- Writes are fast, but not transactional
- Reads are fast, but slightly stale
- Updates are actually delete + reinsert operations

### What Elasticsearch Is Used For

<span style="margin-left: 10px;"></span>
Elasticsearch excels at:
- Full-text search
- Fuzzy matching and relevance scoring
- Faceted navigation
- Aggregations and analytics
- Log and event exploration
- Autocomplete and suggestions

<span style="margin-left: 10px;"></span>
It is not well suited for:
- Strong consistency
- Complex joins
- Multi-row transactions
- High-frequency point updates
- Referential integrity

<span style="margin-left: 10px;"></span>
In production systems, Elasticsearch is almost always a secondary system, derived from a primary data source.

### Near-Real-Time and Eventual Consistency

<span style="margin-left: 10px;"></span>
Elasticsearch is near-real-time, not real-time. There is always a delay between indexing and visibility. This delay 
is acceptable for search, but disastrous if misunderstood.

<span style="margin-left: 10px;"></span>
Any architecture that assumes:
- Immediate visibility
- Strong consistency
- Exactly-once semantics
 
…will eventually produce incorrect results.

## How Elasticsearch Actually Works

### Data Organization and the Inverted Index

<span style="margin-left: 10px;"></span>
At the lowest level, Elasticsearch relies on Lucene’s inverted index — a data structure optimized for fast full-text 
search. Instead of scanning documents on query time, Elasticsearch pre-processes content into a lookup structure 
that maps every unique term (token) to the documents in which it appears. This inversion of document -> terms to 
term -> documents enables milliseconds-level search even across very large datasets.

<span style="margin-left: 10px;"></span>
For example, a field containing “Elasticsearch tutorial” is tokenized into terms like "elasticsearch" and "tutorial", 
and the inverted index tracks which documents contain those terms.

### What Is an Index?

<span style="margin-left: 10px;"></span>
In Elasticsearch, an index is a logical namespace for storing related documents — conceptually similar to a 
database in RDBMS terms (if it even makes sense). An index is the unit you interact with when performing searches 
and aggregations. Internally, however, an index is not a single monolithic structure; it’s a logical grouping of shards.

<span style="margin-left: 10px;"></span>
Documents in an index are JSON objects. Each document has:
- A set of fields (key/value pairs)
- A unique identifier
- A type-agnostic mapping that determines how each field is indexed and searched

<span style="margin-left: 10px;"></span>
Once data hits the index, it goes through analysis — tokenization and normalization — before being persisted in 
the inverted index.

### Shards

<span style="margin-left: 10px;"></span>
An index is partitioned into shards — the basic units of storage and parallelism. Each shard is itself a full 
Lucene index with its own inverted index and segment files, and shards can be located on different nodes in a cluster.

<span style="margin-left: 10px;"></span>
There are two shard types:
- Primary shards — hold the original data and are the destination for write operations
- Replica shards — copies of primary shards that provide redundancy and read scalability

<span style="margin-left: 10px;"></span>
Shard distribution across nodes achieves three major goals:
1. Horizontal Scalability: More shards can be spread across more nodes, boosting capacity.
2. Fault Tolerance: If a node fails, replicas on other nodes ensure availability.
3. Parallel Query Execution: Queries are executed on all relevant shards in parallel, with the results merged by a 
coordinating node.

<span style="margin-left: 10px;"></span>
Once you create an index, the number of primary shards is fixed. Changing it later requires reindexing. 
Replicas, by contrast, can be adjusted dynamically.

<span style="margin-left: 10px;"></span>
Shard count is one of the most consequential design decisions in Elasticsearch. Every shard is a full 
Lucene index with its own segment files, caches, and metadata. This means shards are not free — they 
consume heap, file descriptors, and CPU scheduling overhead.

<span style="margin-left: 10px;"></span>
Oversharding is one of the most common production failures. Clusters with thousands of tiny shards often 
experience:

- High cluster state update latency
- Excessive heap pressure on master nodes
- Slow shard allocation and recovery
- Increased query fan-out

<span style="margin-left: 10px;"></span>
A practical production rule is:

- Target shard sizes between **10GB and 50GB**
- Keep shard counts **below ~20–40 shards per node**
- Prefer **fewer larger shards** over many small ones

<span style="margin-left: 10px;"></span>
Shard sizing is a capacity planning exercise, not just a data partitioning strategy. 
It determines how efficiently a cluster can scale, recover, and execute queries in parallel.

### Nodes and Their Roles

<span style="margin-left: 10px;"></span>
A node is a single Elasticsearch process (typically on a single machine). Nodes collaborate within a cluster 
and can have specialized roles:
- Master-eligible nodes — manage cluster metadata and orchestrate shard allocation
- Data nodes — actually store shards and perform indexing/search operations
- Ingest nodes — preprocess documents before indexing (e.g., enrich or transform)
- Coordinating (client) nodes — accept client requests and forward them to the relevant shards

<span style="margin-left: 10px;"></span>
Roles can be mixed or dedicated depending on workload and scale. In heavy-production environments, it’s 
common to isolate master nodes to reduce load and prevent instability during high query or indexing throughput.

### Cluster

<span style="margin-left: 10px;"></span>
A cluster is a group of nodes that share the same cluster name and work together. The cluster state — a 
shared data structure that tracks indices, shards, mapping schemas, node membership, and more — is maintained
by the elected master. Any change affecting the topology (e.g., new index, node joining) updates the cluster 
state and propagates it to all nodes.

<span style="margin-left: 10px;"></span>
Clusters allow Elasticsearch to behave as a single logical system rather than a bunch of independent servers.

### Data Flow

#### Indexing
1. A client sends a JSON document to a coordinating node
2. The coordinating node routes it to a specific primary shard
3. The document is analyzed and written to a translog and memory buffer
4. The coordinating node forwards the operation to replica shards
5. After an index refresh, the document becomes searchable

<span style="margin-left: 10px;"></span>
A critical nuance here is that indexing is near-real-time: documents only become visible after a refresh, 
which happens automatically but is tunable.

<span style="margin-left: 10px;"></span>
Indexing documents one at a time is rarely acceptable in production systems. Elasticsearch is optimized for 
batch ingestion using the **Bulk API**, which allows many indexing operations to be processed in a 
single request.

<span style="margin-left: 10px;"></span>
Production systems typically batch documents into groups of hundreds or thousands before sending them to 
Elasticsearch. This approach reduces network overhead and significantly improves throughput.

<span style="margin-left: 10px;"></span>
However, aggressive batching without flow control can overwhelm a cluster. Indexing pipelines must include 
backpressure mechanisms to avoid saturating:

- thread pools
- disk I/O
- translog writes
- refresh cycles

Common production practices include:

- limiting concurrent bulk requests
- dynamically adjusting batch size
- retrying rejected requests
- routing failed operations to a dead-letter queue

A stable indexing pipeline is not defined by maximum throughput, but by **sustained throughput under load without destabilizing the cluster**.

#### Searching
1. A search request reaches any node (often a coordinator)
2. The coordinator identifies which shards (primaries and replicas) need to be queried
3. Queries execute in parallel on each shard
4. Partial results return to the coordinator
5. Results are merged, scored, sorted, and sent back to the client

<span style="margin-left: 10px;"></span>
Because shards act in parallel, overall latency is bounded by the slowest responding shard.

#### Replication and High Availability

<span style="margin-left: 10px;"></span>
Replica shards exist on different nodes from their primaries. This protects the cluster against machine 
failures — if a node with primaries fails, its replicas can be promoted. Replica shards also serve read 
operations, increasing throughput for search and retrieval.

## The Production Contract: SLOs, Consistency, and “Search Truth”

<span style="margin-left: 10px;"></span>
Before Elasticsearch becomes useful in production, it must be demoted. It is not the source of truth. It is 
not a transactional store. It is a derived system whose correctness is defined by explicit service-level 
objectives, not by database guarantees.

<span style="margin-left: 10px;"></span>
Production systems that succeed with Elasticsearch do so because they define — early and explicitly — what 
Elasticsearch is allowed to be wrong about.

### Elasticsearch Is Not a Consistent Read Model

<span style="margin-left: 10px;"></span>
Elasticsearch provides near-real-time visibility, not linearizability and not read-your-writes semantics. 
There is always a window where:
- A document has been acknowledged as indexed
- The same document is not yet visible to search
- Different replicas may expose different views

<span style="margin-left: 10px;"></span>
This is not a bug. It is a deliberate tradeoff that enables high write throughput and low-latency search.

<span style="margin-left: 10px;"></span>
Any architecture that assumes:
- Immediate visibility after indexing
- Monotonic reads
- Exactly-once processing

<span style="margin-left: 10px;"></span>
…will eventually violate correctness under load, node failure, or shard relocation.

### Defining the Search Contract

<span style="margin-left: 10px;"></span>
In production, Elasticsearch must operate under an explicit search contract, typically defined per use case:

| Use Case              | Allowed Staleness | Correctness Expectation  |
| --------------------- | ----------------- | ------------------------ |
| Full-text search      | Seconds           | Eventually consistent    |
| Autocomplete          | Sub-second        | Best-effort              |
| Analytics dashboards  | Minutes           | Approximate              |
| User-facing filtering | Seconds           | Deterministic but stale  |
| Compliance / auditing | Never             | Do not use Elasticsearch |

<span style="margin-left: 10px;"></span>
This contract drives:
- Refresh interval configuration
- Indexing pipeline design
- Retry and reconciliation strategies
- User-facing guarantees (or lack thereof)

<span style="margin-left: 10px;"></span>
If you cannot write this table for your system, Elasticsearch is being used without a safety net.

### System of Record and Repairability

<span style="margin-left: 10px;"></span>
In well-designed systems, Elasticsearch is always downstream from a system of record:
- Relational database
- Event log
- Immutable data lake

<span style="margin-left: 10px;"></span>
This is not just a data flow choice — it is an operational escape hatch.

<span style="margin-left: 10px;"></span>
When (not if) Elasticsearch drifts:
- Documents can be replayed
- Indices can be rebuilt
- Mappings can be corrected
- Relevance models can be retrained

<span style="margin-left: 10px;"></span>
If rebuilding an index is considered catastrophic, the architecture is already fragile.

### Read-After-Write: When You Actually Need It

<span style="margin-left: 10px;"></span>
Occasionally, systems do require immediate visibility — for example, after a user edits content and expects 
to see it reflected instantly.

<span style="margin-left: 10px;"></span>
In these cases, production systems do not change Elasticsearch’s consistency model. Instead, they:
- Read directly from the primary datastore
- Overlay recent writes in-memory
- Delay search exposure explicitly
- Or accept temporary inconsistency in UI

<span style="margin-left: 10px;"></span>
Forcing Elasticsearch to behave like a transactional database (e.g., via frequent refreshes or synchronous 
indexing) trades correctness illusions for throughput collapse.

### The Hidden Failure Mode: False Confidence

<span style="margin-left: 10px;"></span>
The most dangerous failure mode in Elasticsearch is not downtime — it is silent incorrectness.

<span style="margin-left: 10px;"></span>
Search results that are:
- Slightly stale
- Incompletely indexed
- Missing edge cases

<span style="margin-left: 10px;"></span>
…are often accepted by systems and users without detection.

<span style="margin-left: 10px;"></span>
This is why production Elasticsearch is designed around:
- Rebuildability
- Observability
- Explicit correctness boundaries

<span style="margin-left: 10px;"></span>
Once those boundaries are clear, Elasticsearch becomes an extremely powerful component — not because it 
is always right, but because it is predictably wrong in controlled ways.

## Mapping Strategy: How You Avoid Reindexing Yourself Into a Corner

<span style="margin-left: 10px;"></span>
In Elasticsearch, mappings are not a convenience feature — they are your schema. Unlike relational databases, 
this schema is not enforced at write time in a transactional sense, but it is baked permanently into the 
index’s physical structure. Once a field is indexed a certain way, that decision cannot be undone without 
reindexing.

<span style="margin-left: 10px;"></span>
Production systems treat mappings as versioned, reviewed artifacts, not as emergent side effects of incoming 
data.

### Index Lifecycle Management (ILM)

<span style="margin-left: 10px;"></span>
Data rarely has the same value forever. Log data, metrics, and event streams often grow continuously and 
require automated lifecycle management.

<span style="margin-left: 10px;"></span>
Elasticsearch provides **Index Lifecycle Management (ILM)** to automate index transitions across storage 
tiers and retention policies.

<span style="margin-left: 10px;"></span>
Typical lifecycle phases include:

- **Hot phase** – active indexing and frequent queries
- **Warm phase** – less frequent queries, optimized storage
- **Cold phase** – infrequent access, cheaper hardware
- **Delete phase** – automatic removal after retention expires

<span style="margin-left: 10px;"></span>
ILM policies can automatically trigger actions such as:

- index rollover when size thresholds are reached
- shard shrinking
- segment merging
- index deletion

<span style="margin-left: 10px;"></span>
Without lifecycle management, clusters accumulate historical data indefinitely and eventually become 
unstable due to excessive shard counts and storage pressure.

## Query Cost and Cluster Stability

<span style="margin-left: 10px;"></span>
Not all queries are equal. Some query patterns are extremely expensive and can destabilize clusters under load.

<span style="margin-left: 10px;"></span>
Production Elasticsearch deployments explicitly control or prohibit queries such as:

- leading wildcard searches (`*term`)
- regular expression queries
- large cardinality aggregations
- script-based scoring
- unbounded nested queries

<span style="margin-left: 10px;"></span>
These operations may trigger full index scans or large in-memory structures that significantly increase 
CPU usage and heap pressure.

<span style="margin-left: 10px;"></span>
In multi-tenant systems or public APIs, query complexity is often restricted through:

- server-side query templates
- query validation layers
- request timeouts
- rate limiting

<span style="margin-left: 10px;"></span>
A single poorly constructed query can consume more resources than thousands of normal queries. For this 
reason, production systems treat query design as an operational concern, not just an application concern.

### Deep Pagination and Why It Breaks Search

<span style="margin-left: 10px;"></span>
Elasticsearch supports pagination using the `from` and `size` parameters, but this mechanism becomes 
increasingly expensive for deep result pages.

<span style="margin-left: 10px;"></span>
When a query requests results far into a result set (for example `from=10000`), every shard must still 
compute and sort all preceding results. This causes large memory allocations and unnecessary work across 
the cluster.

For deep pagination scenarios, production systems instead rely on:

- `search_after` for cursor-like pagination
- the `scroll` API for large batch exports
- user interface limits that prevent navigating arbitrarily deep result pages

<span style="margin-left: 10px;"></span>
Deep pagination is rarely meaningful for users and often becomes a hidden performance trap for search clusters.

### Dynamic Mapping

<span style="margin-left: 10px;"></span>
By default, Elasticsearch will infer field types dynamically. This is useful for experimentation and 
catastrophic in production.

<span style="margin-left: 10px;"></span>
Dynamic mapping failures are subtle:
- A field starts as keyword, later becomes text
- Numeric fields oscillate between long and float
- Date formats diverge across producers
- User-generated JSON explodes into thousands of distinct fields

<span style="margin-left: 10px;"></span>
Once indexed, these mistakes cannot be corrected in-place.

<span style="margin-left: 10px;"></span>
Production-grade indices either:
- Disable dynamic mapping entirely, or
- Constrain it with dynamic: strict and explicit templates

<span style="margin-left: 10px;"></span>
If a document fails to index due to an unexpected field, that is a feature, not a bug — it forces schema 
drift to surface early.

### Field Types Are Query Decisions

<span style="margin-left: 10px;"></span>
Every field type encodes assumptions about how the data will be queried.

<span style="margin-left: 10px;"></span>
Common production patterns include:
- text for full-text search
- keyword for filtering, sorting, and aggregations
- Multi-fields (text + keyword) when both behaviors are required
- scaled_float for monetary or precision-sensitive numeric data
- date with explicit formats, never defaults

<span style="margin-left: 10px;"></span>
Choosing a field type is not about storage — it is about:
- Whether the field participates in relevance scoring
- Whether it can be aggregated efficiently
- Whether it can be sorted without loading fielddata into heap

<span style="margin-left: 10px;"></span>
These decisions directly affect query latency, heap pressure, and cluster stability.

### Analyzers

<span style="margin-left: 10px;"></span>
Analyzers determine what a field means in search. Tokenization, normalization, stemming, synonym expansion — 
these are not technical details; they are product semantics.

<span style="margin-left: 10px;"></span>
Production systems:
- Explicitly define analyzers per field
- Version analyzer configurations
- Avoid global analyzer reuse across unrelated fields

<span style="margin-left: 10px;"></span>
Changing an analyzer changes the inverted index. This means:
- Existing documents do not retroactively benefit
- Queries become inconsistent across old and new data
- A full reindex is required for correctness

<span style="margin-left: 10px;"></span>
Treat analyzer changes with the same gravity as a schema migration in a relational database.

### Field Explosion and Cluster State Pressure

<span style="margin-left: 10px;"></span>
Every mapped field contributes to:
- Cluster state size
- Heap usage on master nodes
- Mapping propagation latency

<span style="margin-left: 10px;"></span>
Field explosion — often caused by unbounded user input or polymorphic JSON blobs — is one of the fastest 
ways to destabilize a cluster.

<span style="margin-left: 10px;"></span>
Production mitigations include:
- Flattening or denormalizing data intentionally
- Using flattened fields for schemaless blobs
- Rejecting documents with excessive field counts
- Separating “searchable” data from “stored” data

<span style="margin-left: 10px;"></span>
If your cluster state grows faster than your data volume, mapping design is already broken.

### Mappings as Versioned Infrastructure

<span style="margin-left: 10px;"></span>
In production, mappings are:
- Stored alongside application code
- Reviewed like API contracts
- Applied through index templates
- Versioned with explicit lifecycle rules

<span style="margin-left: 10px;"></span>
A common pattern is to treat each breaking mapping change as a new index version:

```
products_v12
products_v13
```

<span style="margin-left: 10px;"></span>
This enables:
- Zero-downtime migrations
- Side-by-side validation
- Safe rollback via alias switching

<span style="margin-left: 10px;"></span>
If your deployment process cannot create a new index with a new mapping at will, Elasticsearch is already 
constraining your delivery velocity.

### The Core Rule

<span style="margin-left: 10px;"></span>
You can evolve queries freely.
You can evolve documents carefully.
You cannot evolve mappings without reindexing.

<span style="margin-left: 10px;"></span>
Production Elasticsearch systems succeed not because they avoid reindexing — but because they design for 
it from day one.

## The End

<span style="margin-left: 10px;"></span>
Although I never used Elasticsearch in production, I have a much deeper understanding of how it works and what it is for 
after writing these "notes". I'm aware that true knowledge comes from experience, but I hope this research will help 
me avoid common errors when I eventually do use it in production. If I will ever do it, that is.

<span style="margin-left: 10px;"></span>
I may not ever use Elasticsearch in production, but I always like to understand how things work. I think that 
understanding the inner workings of a technology can help me make better decisions about when and how to use it, 
even if I never end up using it directly.













