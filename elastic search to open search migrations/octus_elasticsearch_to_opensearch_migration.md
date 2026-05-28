# Octus: 85% Cost Reduction — Elasticsearch to Amazon OpenSearch Migration

> Source: AWS Big Data Blog, 26 Nov 2025  
> Authors: Vaibhav Sabharwal, Andre Kurait, Harmandeep Sethi, Serhii Shevchenko, Govind Bajaj, Virendra Shinde, Brian Presley

---

## Conversation 1 — The Problem and Solution Explained Intuitively

### The Problem

Octus was paying a massive bill to run **Elasticsearch on Elastic Cloud** (a third-party managed service). As their data grew, costs ballooned and they had no good way to scale down. The brutal constraint: they **couldn't turn off the service** even for a few minutes — financial intelligence customers need real-time data 24/7.

Traditional migration = take a snapshot, shut down old system, restore to new system, flip the switch. That means **downtime**. Not acceptable here.

---

### The Intuitive Solution: The "Shadow Copy" Trick

Think of it like replacing the engine of a plane **while it's flying**.

```
Phase 1 — Install a spy in the middle
   Client → [Capture Proxy] → Old Elasticsearch
                ↓ (secretly copies every request)
              Amazon MSK (Kafka queue)
```
Every single search/write request gets **secretly recorded** into a Kafka queue. Old system keeps running normally. Zero user impact.

```
Phase 2 — Backfill history
   Old S3 Snapshot → [Reindex-from-Snapshot] → New OpenSearch
```
They took a point-in-time snapshot of all historical data and bulk-loaded it into the new cluster in the background. No load on the live system.

```
Phase 3 — Replay the gap
   Kafka queue → [Traffic Replayer] → New OpenSearch
```
All the live traffic that happened *during* the backfill gets replayed onto the new cluster. Now both clusters are in sync.

```
Phase 4 — Verify, then flip
   Client → Old ES  (confirm new cluster behaves identically)
   Client → New OpenSearch  (single DNS/load balancer change)
```

**Zero downtime.** The cutover is just a load balancer rule change.

---

### Why 85% Cost Reduction?

Two-stage savings:

| Stage | What happened | Saving |
|---|---|---|
| Migration | Moved off Elastic Cloud to AWS-managed OpenSearch | 52% |
| Post-migration | Analyzed real usage data, right-sized the cluster | +33% |

They had been **over-provisioned** for years on Elastic Cloud. AWS gave them the data to see exactly what they actually needed, then they shrank to fit.

---

### The Hidden Complexity They Had to Solve

- **Version mismatch**: Source was ES 7.17, target was OpenSearch 1.3 — APIs differ slightly, so they had to rewrite clients in PHP, Python, and even R (no official OpenSearch R client existed, so they built a custom one)
- **Metadata incompatibility**: Timestamp field mappings broke — they wrote a custom JavaScript transform inside the Migration Assistant to auto-fix mappings across dozens of indices
- **Pre-cleanup**: Before migrating, they deleted stale indices and oversized documents — reducing the data volume and making the backfill faster and cheaper

---

## Conversation 2 — Deep Dive Q&A

### Q: How does Capture Proxy get the old data — is it like ETL?

**No — it's a wiretap, not ETL.**

```
Client → [Capture Proxy] → Elasticsearch  (normal response)
                ↓
         copies the HTTP request into Kafka
```

It does **not** go fetch old data. It purely intercepts **new incoming HTTP requests** from the moment it's deployed. Think of it as a **transparent man-in-the-middle** — the client never knows it's there, Elasticsearch responds normally, but every request gets secretly copied.

Old historical data is handled entirely separately by the S3 snapshot in Phase 2. These two are completely independent streams.

---

### Q: Phase 2 — What is the data format? Is this vector embeddings like Pinecone?

**In Octus's specific case: NO** — these are plain JSON documents (structured financial/credit intelligence records). Traditional keyword/full-text search (BM25), not semantic vector search.

But since you know Pinecone, here's the full picture of what Elasticsearch/OpenSearch can store:

```
Pinecone concept         →  OpenSearch equivalent
─────────────────────────────────────────────────
Namespace                →  Index
Vector (dense float[])   →  knn_vector field
Metadata                 →  Regular JSON fields alongside the vector
Sparse vector            →  neural_sparse field (keyword-style)
Hybrid search            →  Yes, OpenSearch supports both simultaneously
```

An Elasticsearch document looks like this (with vectors):
```json
{
  "company": "Octus Corp",
  "credit_rating": "BBB",
  "embedding": [0.12, 0.87, 0.34, ...]
}
```

**Is migration just index creation + bulk ingest?**

Mostly yes, but with friction:

```
Step 1 — Metadata migration
  ES mapping:  { "embedding": { "type": "dense_vector", "dims": 768 } }
  OS mapping:  { "embedding": { "type": "knn_vector",   "dimension": 768 } }
                                           ↑ different field type name!

Step 2 — Reindex (bulk ingest)
  Read documents from S3 snapshot → transform if needed → POST to OpenSearch _bulk API
```

The timestamp incompatibility Octus hit is exactly this kind of mapping mismatch — they wrote a JS transform to auto-fix it across all indices before ingestion.

---

### Q: What is actually stored in Kafka? Is it the real data?

**Kafka stores the raw HTTP requests, not document copies.**

```
What gets written to Kafka:
─────────────────────────────────────────────────────
PUT /my-index/_doc/id-123
{ "company": "Octus", "rating": "BBB", "ts": "2024-01-01" }

DELETE /my-index/_doc/id-456

POST /my-index/_update/id-789
{ "doc": { "rating": "BB" } }
```

It is **not** storing the full dataset — just the stream of **write operations** (creates, updates, deletes) since the proxy was turned on. Read requests (searches) are also captured for verification but aren't needed for data sync.

Why Kafka:
- **Ordered** — events must replay in the same sequence (update then delete, not delete then update)
- **Durable** — nothing lost even if the replayer is slow or restarts
- **Decoupled** — backfill can take days; Kafka buffers everything that happened meanwhile

```
Timeline:
Day 0   → Capture proxy ON, Kafka starts recording writes
Day 1-3 → S3 snapshot covers everything up to Day 0
Day 4   → Reindex-from-Snapshot finishes (data up to Day 0 is in OpenSearch)
Day 4   → Replayer replays Day 0→Day 4 from Kafka
Day 4+  → Now both clusters are in sync in real-time
```

The snapshot covers the **gap before the proxy was on**. Kafka covers **everything after**. Together they have 100% coverage.

---

### Q: Phase 4 Cutover — is it a load balancer between OpenSearch and Elasticsearch?

**Yes, exactly.**

```
Before cutover:
  Client → ALB → Capture Proxy → Elasticsearch

After cutover:
  Client → ALB → OpenSearch
```

The ALB **target group** just gets switched. Clients hit the same hostname/IP — they never know. But there's one extra layer of work Octus had to do: **swap the client libraries** in the application code (elasticsearch-py → opensearch-py, etc.), because the API has minor differences. That code change was deployed at the same time as the ALB switch, coordinated as a single release with no downtime.

---

## Architecture Diagram Reference

| Step | Component | What it does |
|---|---|---|
| 1 | Client Traffic → Source Cluster | Normal traffic flow, unchanged |
| 2 | ALB → Capture Proxy → MSK | Wiretap: copies all requests to Kafka |
| 3 | S3 Snapshot → RFS Tasks → Target | Bulk historical data load |
| 4 | MSK → Replayer → Target | Replay live writes captured during backfill |
| 5 | CloudWatch + EFS | Monitor both clusters, compare behavior |
| 6 | ALB target switch | Cutover — clients redirected to OpenSearch |

**Target cluster**: Amazon OpenSearch Service 1.3  
**Source cluster**: Elasticsearch 7.17 on Elastic Cloud
