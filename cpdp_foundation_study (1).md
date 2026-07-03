# Architecture Study — Data Foundation / Card Payment Data Platform

**Contract-Based Ingestion, Iceberg Raw Lake, MongoDB Feeding & Organized Model (v1 → v2)**

| | |
|---|---|
| **Version** | v1.0 |
| **Date** | July 2026 |
| **Owner** | Architecture — CPDP |
| **Labels** | `ingestion` `kafka` `iceberg` `mongodb` `organized-layer` `data-contract` |

---

## Table of Contents

- [0. Context, Scope and Guiding Principles](#0-context-scope-and-guiding-principles)
- [1. Ingestion](#1-ingestion)
  - [1.1 The Ingestion Contract — and Its Distinction from the Organized Model](#11-the-ingestion-contract--and-its-distinction-from-the-organized-model)
  - [1.2 Pipeline 1 — Raw Storage: Kafka (Avro/JSON) → Parquet / Iceberg](#12-pipeline-1--raw-storage-kafka-avrojson--parquet--iceberg)
    - [1.2.1 From Unit Messages to Parquet Files](#121-from-unit-messages-to-parquet-files--the-problem-and-its-resolution)
    - [1.2.2 Iceberg Commit Mechanics and Catalog Updates](#122-iceberg-commit-mechanics-and-catalog-updates)
    - [1.2.3 Iceberg Partitioning Strategy](#123-iceberg-partitioning-strategy)
    - [1.2.4 Table Maintenance (Compaction, Snapshots)](#124-table-maintenance--not-optional)
  - [1.3 Pipeline 2 — Feeding the MongoDB Transactional Database](#13-pipeline-2--feeding-the-mongodb-transactional-database-organized-model)
    - [1.3.1 Choosing Kafka Production Keys](#131-choosing-kafka-production-keys--an-architectural-decision-not-a-detail)
    - [1.3.2 Sizing the Number of Partitions](#132-sizing-the-number-of-partitions--large-from-creation)
    - [1.3.3 Correlation Secured at Ingestion — Never During Processing](#133-correlation-secured-at-ingestion--never-during-processing)
    - [1.3.4 Desynchronized Consolidation and Proactive Orphan Detection](#134-desynchronized-consolidation-and-proactive-orphan-detection)
- [2. The Organized Model](#2-the-organized-model)
  - [2.1 Semantic Independence from Producers](#21-semantic-independence-from-producers)
  - [2.2 Current State — Simplified v1 Model](#22-current-state--simplified-v1-model)
  - [2.3 Target v2 Model — the Acquiring Business](#23-target-v2-model--the-acquiring-business)
  - [2.4 v1 → v2 Migration Strategy Without Service Disruption](#24-v1--v2-migration-strategy-without-service-disruption)
- [3. Summary of Architecture Principles](#3-summary-of-architecture-principles)

---

## 0. Context, Scope and Guiding Principles

The Card Payment Data Platform (CPDP, the "data foundation") centralizes acquiring card payment data — transactions, authorizations, captures, remittances, chargebacks, fraud — produced by heterogeneous systems (NEPTING, K2/Partecis via BCEF-IT/DHF, E-Stream, Computop, batch files on IBM COS) and makes it available to consumers: Accepta Vision Portal, the Axepta reporting platform, regulatory and accounting extractions.

This study lays the foundations of the platform across two areas: **ingestion** (contracts, raw pipeline to Iceberg, transactional pipeline to MongoDB, correlation) and the **organized model** (semantic independence, v1 → v2 trajectory). The **exposition and large-volume report extraction** area is covered in a dedicated study.

> **Founding principle — two representations, one single truth**
>
> The same data exists in two physical representations, fed in parallel from Kafka and optimized for opposite usage profiles:
>
> - The **Iceberg/Parquet Raw Lake**: immutable, exhaustive capture of events as received. Eternal source of truth, replayable, never modified. The bedrock for analytics, reconstructions and migrations.
> - The **MongoDB organized model**: structured, correlated representation in the business language of acquiring. Optimized for hot reads and exposition.
>
> Each zone is fed **by projection from the stream**, never by lateral copy between served zones. This decoupling is what will make the v1 → v2 migration of the organized model possible without drama (see §2.4).

```
                          PRODUCERS
     NEPTING (ISN/Apigee)   K2/Partecis via DHF   E-Stream   Computop   IBM COS (batch)
            │                     │                  │           │           │
            └───────────┬─────────┴──────────┬───────┴───────────┴───────────┘
                        ▼                    ▼
              ╔══════════════════════════════════════════╗
              ║   INGESTION CONTRACTS (Avro / Registry)   ║   ← contractual boundary
              ╚══════════════════════════════════════════╝
                        │
                 KAFKA — ingestion topics
                 (key = correlation identity, 64–128 partitions)
                        │
        ┌───────────────┴───────────────────┐
        │  consumer group:                   │  consumer group:
        │  raw-lake-writer                   │  organized-writer
        ▼                                    ▼
  ┌──────────────────────┐        ┌───────────────────────────┐
  │  RAW LAKE            │        │  ORGANIZED MODEL          │
  │  Iceberg + Parquet   │        │  MongoDB                  │
  │  (S3 / COS / HDFS)   │        │  transactions, remittances│
  │  immutable,          │──────▶ │  chargebacks (v1 → v2)    │
  │  replayable          │ replay │  idempotent upserts       │
  │  partition: day      │        │                           │
  └──────────────────────┘        └───────────────────────────┘
        │                                    │
        ▼                                    ▼
   Analytics / Reporting              Exposition APIs
   (Starburst — dedicated study)      (Vision Portal — dedicated study)
```

---

## 1. Ingestion

### 1.1 The Ingestion Contract — and Its Distinction from the Organized Model

The ingestion contract is a **formal, versioned, testable agreement** between each producer and the platform. It is not a documentation page: it is a code artifact (Avro schemas in a Schema Registry, executable quality rules) whose violation is detected automatically, at the boundary, before any ingestion takes place.

#### What the contract specifies

| Dimension | Contractual content |
|---|---|
| **Schema** | Versioned Avro schema in the Schema Registry (one subject per topic). Transport format: binary Avro (recommended) or JSON validated against the schema. Evolution policy: `BACKWARD_TRANSITIVE` — a producer can never publish a change that would break existing consumers. |
| **Identities** | Globally unique `event_id` (idempotency), `event_type` (AUTHORIZATION, CAPTURE, REMITTANCE, CHARGEBACK, FRAUD…), `source_system`. |
| **Correlation keys** | **Mandatory from production onward**: transaction identifier / Correlation ID (authorization ↔ capture link), remittance reference, file reference (stamping references). An event without its correlation keys is rejected at the boundary — see §1.3.3; this is the keystone of the pipeline. |
| **Temporality** | Contractual distinction between `business_time` (the business time of the event) and `produced_at` (the publication time). All the platform's temporal logic anchors on `business_time`. |
| **Quality** | Completeness of mandatory fields, uniqueness of `event_id`, freshness (SLA of publication latency after the business event), publication order per key. |
| **Error channel** | Behavior on rejection: dedicated DLQ per producer, non-compliance report, remediation loop (see §1.3.4). |

#### The fundamental distinction: ingestion contract ≠ organized model

The ingestion contract captures data **in the producer's semantics**: the E-Stream vocabulary (ISO 20022-inspired), K2 codes, NEPTING fields. The organized model expresses data **in the business language of acquiring**, unique and independent from producers (see §2.1). Between the two lies an explicit translation layer — the **ingestion mapping** — itself versioned and tested.

```
   PRODUCER semantics                TRANSLATION                BUSINESS semantics
  ┌────────────────────────┐   ┌──────────────────────┐   ┌─────────────────────────┐
  │  INGESTION CONTRACT    │   │  INGESTION MAPPING   │   │   ORGANIZED MODEL       │
  │  (Avro, per producer)  │──▶│  (anti-corruption    │──▶│   (acquiring pivot,     │
  │  E-Stream / K2 / NEPT. │   │   layer, versioned,  │   │    unique, versioned)   │
  │                        │   │   golden tests)      │   │                         │
  └────────────────────────┘   └──────────────────────┘   └─────────────────────────┘
```

> ⚠️ **Anti-pattern to ban — semantic leakage**
>
> If producer fields and concepts leak directly into the organized model ("E-Stream field 422", "K2 return code"), the model becomes the **union of producer formats**: every new producer deforms it, every upstream evolution breaks it, and consumers must know the source systems to interpret it. The contract absorbs producer semantics; the mapping translates it; the organized model ignores it. Provenance is preserved only in technical lineage fields (`source_system`, `source_message_ref`, `ingestion_schema_version`), never in business semantics.

#### Contract governance

- **The contract is code**: `.avsc` files in Git, mandatory review, CI validating compatibility against the Registry before any producer deployment.
- **Golden samples**: each producer provides a reference message set (nominal cases + edge cases). These samples feed the automated tests of the ingestion mapping.
- **Evolution**: any schema change follows the procedure — proposal, platform + producer review, publication of a new compatible version, planned deprecation of removed fields.
- **The Business Analyst facilitates, the CI validates**: the BA drives contract drafting with the producer and engineering; compliance is no longer a matter of Confluence prose but of tests executed on every build.

### 1.2 Pipeline 1 — Raw Storage: Kafka (Avro/JSON) → Parquet / Iceberg

Objective: capture **all** events, as received, in immutable, low-cost, SQL-queryable, replayable storage. The Raw Lake is the platform's safety net: any projection (including the organized model itself) can be rebuilt from it. File format: **Parquet**; table format: **Apache Iceberg**.

#### 1.2.1 From Unit Messages to Parquet Files — the Problem and Its Resolution

Parquet is a **block-based columnar format**: it is designed to be written in large batches and read by columns, not written message by message. Writing one file per event would produce millions of "small files" that destroy read performance (each file = an S3 open/close, metadata, a footer) and make Iceberg metadata explode. Going from the unit Kafka stream to Parquet files therefore relies on an **accumulation buffer**:

```
  Kafka (unit Avro messages)
      │  continuous poll()
      ▼
  ┌──────────────────────────────────────────────────────┐
  │  WRITER (Kafka Connect / Flink)                      │
  │                                                      │
  │  1. Avro deserialization (Schema Registry)           │
  │  2. Routing to the target Iceberg table/partition    │
  │     (by event_type, by business_time day)            │
  │  3. IN-MEMORY ACCUMULATION / open files              │
  │     one Parquet writer open per (table, partition)   │
  │  4. FLUSH when threshold reached:                    │
  │     - target file size (128–512 MB)                  │
  │     - OR end of commit interval (2–5 min)            │
  │  → closed Parquet data files, ready to be committed  │
  └──────────────────────────────────────────────────────┘
      │
      ▼  ICEBERG COMMIT (atomic, see §1.2.2)
  Iceberg table — new files become visible all at once
```

Two reference implementations:

| Option | How it works | When to choose it |
|---|---|---|
| **Kafka Connect — Iceberg Sink Connector** | Connect workers consume, buffer and write the Parquet data files. A **commit coordinator** aggregates the files produced by all workers and performs a single Iceberg commit at a fixed interval (`iceberg.control.commit.interval-ms`, typ. 120,000–300,000 ms). **Kafka offsets are recorded in the snapshot properties**: on crash, the connector resumes from the last commit → exactly-once semantics. Supports **multi-table fan-out** (route each event_type to its own table) and automatic table creation with schema evolution. | **Recommended for raw**: no transformation, declarative configuration, no code to maintain. |
| **Flink — Iceberg Sink** | Files are written continuously; the Iceberg commit is aligned on **Flink checkpoints** (end-to-end exactly-once via two-phase commit). Latency and commit frequency driven by the checkpoint interval. | If in-flight enrichment or normalization becomes necessary before raw (to be avoided on principle: raw must stay raw). |

> ⚠️ **Rule — commit frequency is a freshness / small-files trade-off**
>
> Committing every 2 to 5 minutes yields a lake that is fresh within minutes — largely sufficient for an analytical raw layer — while producing healthy file sizes. Never chase "second-level freshness" on the lake: that is the role of the MongoDB pipeline (§1.3). Committing per message or per tiny batch is the classic mistake that renders an Iceberg table unusable within six months.

#### Raw envelope — table schema

Every raw table carries a standard envelope, identical for all event types, plus the payload in the producer's semantics (the raw layer translates nothing):

```
event_id            string    — unique identity (idempotency, deduplication)
event_type          string    — AUTHORIZATION | CAPTURE | REMITTANCE | CHARGEBACK | FRAUD | …
source_system       string    — ESTREAM | K2 | NEPTING | COMPUTOP | BATCH_COS
schema_version      string    — version of the applied ingestion contract
business_time       timestamp — business time (partitioning column)
produced_at         timestamp — Kafka publication time
ingested_at         timestamp — lake write time
correlation         struct    — { transaction_id, correlation_id, remittance_ref, file_ref, siren, rib, store_id }
kafka               struct    — { topic, partition, offset }  — replay traceability
payload             struct    — the producer's message, complete, untranslated
```

#### 1.2.2 Iceberg Commit Mechanics and Catalog Updates

Understanding Iceberg's internal mechanics is essential for sizing maintenance. An Iceberg table is a **tree of immutable metadata** pointed to by the catalog:

```
  CATALOG (REST / Nessie / Glue / Hive Metastore)
      │  stores only ONE thing per table:
      │  the pointer to the current metadata.json
      ▼
  metadata.json (vN) ── schema, partition spec, snapshot history
      │
      ▼
  manifest-list (of the current snapshot) ── list of manifests + partition stats
      │
      ├──▶ manifest-file A ── list of data files + min/max stats per column
      │        ├──▶ data-file-001.parquet
      │        └──▶ data-file-002.parquet
      └──▶ manifest-file B
               └──▶ data-file-003.parquet

  ONE COMMIT =
   1. write the new Parquet data files              (invisible to readers)
   2. write a new manifest referencing them          (invisible)
   3. write a new manifest-list = snapshot N+1       (invisible)
   4. write metadata.json vN+1                       (invisible)
   5. atomic CAS on the catalog:
      "replace pointer vN with vN+1 if the current one is still vN"
      → optimistic concurrency: on conflict, re-read + retry
      → readers see the old snapshot OR the new one, never a partial state
```

Practical consequences:

- **Free snapshot isolation**: a Starburst query started during a commit reads a consistent snapshot end to end.
- **Native time travel**: every commit is a timestamped snapshot; the table can be queried "as it was" at a past instant — invaluable for audit and accounting reconciliation.
- **The catalog is critical but lightweight**: it only performs the pointer CAS. Recommendation: a **REST catalog** (the 2025+ interoperability standard, supported by Starburst, Spark, Flink, Kafka Connect) backed by an HA store.
- **Connector exactly-once**: since Kafka offsets are stored in the snapshot properties, offset-commit and data-commit are **the same atomic act** — no double-write window, no loss.

#### 1.2.3 Iceberg Partitioning Strategy

Iceberg practices **hidden partitioning**: the partition is declared as a *transformation* of a column (`days(business_time)`), files are physically organized according to this transformation, and queries filtering on `business_time` benefit from pruning **without ever manipulating a partition column**. This is a decisive advantage over Hive partitioning (no technical `dt=2026-07-01` column to know about, no possible query mistake).

| Decision | Recommended choice | Justification |
|---|---|---|
| **Table granularity** | **One table per event type**: `authorizations_raw`, `captures_raw`, `remittances_raw`, `chargebacks_raw`, `fraud_raw` (connector fan-out by `event_type`). | Heterogeneous payload schemas; direct single-movement extractions (lesson from the "authorizations only" need — see exposition study); differentiated volumes and retentions. |
| **Primary partition** | `days(business_time)` | All analytical queries and backfills filter by business period. Aligned with the business-day logic of remittances. |
| **Sub-partition** | **None at start.** No `identity(source_system)`, no `bucket()` by default. | Over-partitioning is mistake #1: it multiplies small files (days × sources × buckets). The manifests' per-column min/max stats already provide effective pruning on `source_system` or `store_id`. |
| **Evolution** | If a massive read pattern later requires it: `bucket(transaction_id, 32)` added via **partition evolution**. | Iceberg evolves the partition spec **through metadata, without rewriting history**: old data keeps the old spec, new data adopts the new one. A reversible decision — hence not to be taken prematurely. |
| **Intra-file sorting** | `write.sort-order` = (business_time, store_id) | Improves compression and fine-grained Parquet page pruning for merchant filters. |

#### 1.2.4 Table Maintenance — Not Optional

A continuous stream mechanically produces files smaller than optimal and an accumulation of snapshots. Maintenance is a **component of the architecture**, scheduled from day one:

| Procedure | Role | Recommended cadence |
|---|---|---|
| `rewrite_data_files` | Compaction: merges small files toward the target size (256–512 MB), applies the sort-order. | Daily, on the D-1 partition (closed partitions no longer change). |
| `expire_snapshots` | Purges old snapshots and the files they were the only ones referencing. Bounds time travel. | Daily — 7-day snapshot retention (long-term replay is done on the current state, not on snapshots). |
| `rewrite_manifests` | Reorganizes manifests for optimal pruning when they fragment. | Weekly. |
| `remove_orphan_files` | Deletes files written but never committed (writer crashes). | Weekly, with a safety guard > 3 days. |

> ✅ **What the Raw Lake brings to the rest of the study**
>
> Total replayability: the MongoDB pipeline can be rebuilt, a mapping bug can be fixed then replayed, and above all — this is the central argument of §2.4 — **the v2 of the organized model can be fed by re-projection from raw**, rather than by a database-to-database migration that would carry over v1's defects.

### 1.3 Pipeline 2 — Feeding the MongoDB Transactional Database (Organized Model)

This pipeline transforms the event stream into organized-model documents: correlated upserts into MongoDB collections, at high throughput, with a freshness requirement far stricter than the lake's (seconds to minutes). Its performance rests on three decisions taken **upstream of the consumer**: the production key, the number of partitions, and correlation carried by the event.

#### 1.3.1 Choosing Kafka Production Keys — an Architectural Decision, Not a Detail

The Kafka message key determines the assigned partition, hence **processing order** and **locality**: all messages with the same key arrive ordered on the same partition and are processed by the same consumer. This property is what enables MongoDB writes without prior reads (§1.3.3).

| Key candidate | Verdict | Analysis |
|---|---|---|
| `transaction_id` / `correlation_id` | ✅ **RECOMMENDED** | All movements of the same transaction (authorization, capture, clearing, chargeback) converge, ordered, onto the same partition → the consumer sees a transaction's lifecycle sequentially, and upserts on the target document never compete across partitions. Very high cardinality (tens of millions) → statistically uniform distribution. |
| `store_id` / `merchant_id` | ❌ **TO BAN** | Creates **hot partitions**: a large merchant (Castorama-like, tens of thousands of transactions/day) concentrates its traffic on a single partition, which becomes the bottleneck of the entire pipeline while others sit empty. |
| Null key (round-robin) | ❌ **TO BAN** | Perfect distribution but **no ordering guarantee**: the capture may be processed before the authorization by two concurrent consumers, with two simultaneous upserts on the same document → race conditions on MongoDB. |
| Remittances (special case) | 🔑 key = `remittance_ref` | Remittances being independent objects (see §2.3), their topic is keyed on the remittance reference; the link to transactions is carried by the data, not by partitioning. |

> 🚨 **Contractual requirement toward producers**
>
> The production key is **part of the ingestion contract** (§1.1). A producer publishing with a wrong key — or no key — silently breaks ordering and locality: the defect only shows under load, as inconsistent documents. The required key, its provenance in the payload and its cardinality are therefore validated at contract review, and monitored (message distribution per partition).

#### 1.3.2 Sizing the Number of Partitions — Large, from Creation

The number of partitions caps the maximum consumer parallelism (beyond it, consumers sit idle) and is decided **at topic creation**. Sizing rules:

```
Sizing throughput:            sustained peak × replay factor (backlog replay) ≈ 3–5×
Prudent capacity/partition:   5–10 MB/s or a few thousand msg/s (depending on message size)
Consumer parallelism:         max active consumers = number of partitions

CPDP example — enriched transactions topic:
  estimated peak 2,000 msg/s × replay 4× = 8,000 msg/s target
  ÷ ~500 msg/s of MongoDB processing per consumer thread (batched upserts)
  → 16 consumers minimum → 64 partitions retained (×4 growth margin)

Recommendation: 64 partitions minimum on transactional topics,
128 on the highest-volume topics (authorizations).
```

> ⚠️ **Why oversize from the start — later increases break key assignment**
>
> Increasing the partition count of an existing topic changes the result of `hash(key) % partition_count`: events of the same transaction, until then routed to partition 12, now go to partition 47 — **while older events of the same key are still pending on partition 12**. Per-key ordering is transiently broken, precisely the guarantee the whole pipeline rests on. The only sound strategy: size large at creation (the cost of a surplus partition is negligible), and treat any later repartitioning as a full migration (new topic + controlled cutover).

#### 1.3.3 Correlation Secured at Ingestion — Never During Processing

This is the pipeline's most structuring decision. Two philosophies stand opposed:

```
  ── ANTI-PATTERN: correlation DURING PROCESSING (blocking lookup) ──────────────

  for each consumed event:
      parent = mongodb.findOne({ ... })        ← UNIT, BLOCKING network call
      if parent found: attach, write
      else: retry later / set aside

  Measurable consequences:
   · ~1–3 ms RTT per lookup × millions of events/day
   · consumer throughput capped by MongoDB latency, not by Kafka
   · lag accumulates from the first database slowdown (backpressure)
   · on backlog recovery: collapse (every replayed event re-pays the lookup)

  ── TARGET PATTERN: correlation AT INGESTION (carried by the event) ────────────

  the contract (§1.1) guarantees the event CARRIES its keys:
      transaction_id, correlation_id, remittance_ref, file_ref

  the consumer then writes BLIND, with no prior read whatsoever:
      bulkWrite(ordered=false) of updateOne(
         filter : { _id: transaction_id },
         update : { $setOnInsert: {skeleton}, $set: {fields},
                    $push / positional $set : movement },
         upsert : true )

   · zero reads in the hot path — throughput is that of batched bulkWrite
   · idempotent and replayable (event_id + version guards like batch_session)
   · per-key ordering (§1.3.1) eliminates races between consumers
```

The case of **out-of-order** events (a capture arriving before its authorization, a chargeback before the transaction exists) is absorbed by the same mechanism: the `$setOnInsert` upsert creates a **skeleton document** carrying the identity and the received movement; the later arrival of the parent completes the skeleton. At no point does the consumer wait, re-read, or block.

> ℹ️ **Contractual treatment of correlation with producers**
>
> Correlation thus becomes a **producer obligation**, not a platform service: E-Stream provides the Correlation ID linking authorization and capture; stamping references link transaction, remittance and file; K2 and NEPTING provide their transaction identities. The platform **verifies** (DLQ rejection if keys are missing) and **assembles** (upserts); it never **guesses**. Any "dynamic" correlation rule that would require finding the parent heuristically is a contract defect to escalate to the producer — not an algorithm to implement in the consumer.

#### 1.3.4 Desynchronized Consolidation and Proactive Orphan Detection

Despite the contracts, reality will produce **orphans**: movements whose parent never arrives (upstream loss), keys present but inconsistent, edge cases during recovery. The principle: these cases are handled **outside the hot path**, by asynchronous processes, and made **visible** rather than silently accumulated.

```
   Hot path (Kafka consumers)                  Asynchronous paths
  ┌────────────────────────────────┐
  │ blind idempotent upserts       │
  │ → complete documents           │
  │ → SKELETON documents           │──┐
  │   (parent not yet arrived)     │  │
  │ → DLQ (contract violated:      │  │   ┌─────────────────────────────────┐
  │   missing keys, schema KO)     │──┼──▶│ CONSOLIDATOR (periodic job)      │
  └────────────────────────────────┘  │   │ · batched scan of skeletons      │
                                      │   │   (index on {status:"stub",      │
                                      │   │    first_seen}) — never unit     │
                                      └──▶│ · attachment attempts            │
                                          │ · re-emission of corrected DLQs  │
                                          └───────────────┬─────────────────┘
                                                          │ beyond thresholds
                                                          ▼
                                          ┌─────────────────────────────────┐
                                          │ PROACTIVE DETECTION              │
                                          │ · metrics: orphan count,         │
                                          │   age of the oldest, correlation │
                                          │   rate — PER PRODUCER            │
                                          │ · thresholds: e.g. orphan > 24h, │
                                          │   rate < 99.9% over 1h           │
                                          │ · NON-CORRELATION REPORT         │
                                          │   notified to the producer       │
                                          │   (event_ids, keys, periods)     │
                                          │ · remediation tracking           │
                                          └─────────────────────────────────┘
```

- **The consolidator** works via indexed, batched sweeps (never unit lookups), at a regular cadence (e.g. every 10 minutes), and never competes with the hot path on the same documents thanks to idempotent upserts.
- **Orphan metrics are producer quality indicators**, published per producer and per event type: they objectify contract compliance and trigger the remediation loop (tooled notification, not artisanal email).
- **Nothing disappears**: a definitive orphan remains queryable (raw + quarantine collection), timestamped, attributable — an audit requirement on financial data.

---

## 2. The Organized Model

### 2.1 Semantic Independence from Producers

The organized model is the **single business language of acquiring**. It describes domain concepts — the transaction and its lifecycle, the movements (authorization, capture, clearing), the remittance, the chargeback, fraud — and not stream formats. E-Stream, K2, NEPTING and tomorrow Computop are **input dialects** that the mapping layer (§1.1) translates into this pivot.

This principle has three concrete consequences:

- **Adding a producer = writing a mapping**, never modifying the model. Onboarding Computop is an integration project (contract + mapping + tests), not a re-modeling project.
- **Multi-source convergence test**: the same transaction observed by K2 and by E-Stream must produce, after mapping, **the same organized document** — lineage fields aside. This test (cross-source golden tests) is the objective criterion of semantic independence, executed in CI.
- **Lineage without pollution**: provenance (`source_system`, `source_message_ref`, `ingestion_schema_version`, Kafka offsets) lives in a technical `_lineage` section of each document. It serves audit and debugging; no business consumer needs to read it to interpret the data.

> ⚠️ **Architect's position — on deriving from the E-Stream transfer model**
>
> The current model was largely deduced from the E-Stream transfer model (itself partially inspired by ISO 20022). A **transport** model is optimized for exchanging messages, not for representing a domain or serving reads: by taking it as a starting point, one inherits messaging constraints in the data core. The choice not to fully adopt a standard (complete ISO 20022, BIAN) for cost reasons is understandable and assumed; on the other hand, drawing on their **conceptual structure** (the vocabulary, the transaction / movements / settlement separation) costs nothing and avoids reinventing a domain the industry has already battle-tested. This is exactly the v2 target: a model designed for the business and the read patterns, *mapped* from E-Stream — not deduced from it.

### 2.2 Current State — Simplified v1 Model

| v1 collection | Content | Identified limits |
|---|---|---|
| `transaction` | "Flattened" transaction document, lifecycle statuses (accepted → authorized → settled), movements partially embedded. | Semantics inherited from the streams; the status-based model overwrites history; single-movement extractions (authorizations only) are inefficient; the transaction/movement distinction is ambiguous for consumers. |
| `remittance` | Remittances (settlement sessions). | Transaction ↔ remittance link unevenly exploited; remittance ↔ file link not exploited. |
| `chargeback` | Chargebacks. | Attachment to the transactional lifecycle not systematic. |

The v2 currently hosts mainly **K2** and **NEPTING** data. The structuring requirement: that v2 serve, **with the same schema**, the data received from K2, NEPTING **and** E-Stream, transparently — the multi-source convergence of §2.1 is not an option, it is the acceptance criterion of v2.

### 2.3 Target v2 Model — the Acquiring Business

The v2 describes the domain properly, drawing the lessons of v1:

```
  ┌───────────────────────────────────────────────┐      ┌──────────────────────────┐
  │  TRANSACTION  (aggregate, collection)         │      │  REMITTANCE (collection)  │
  │  _id = transaction_id (correlation key)       │      │  _id = remittance_ref     │
  │  identity: store_id, merchant, siren, rib     │      │  INDEPENDENT object:      │
  │  consolidated amounts, currency               │      │  its own amounts,         │
  │  key business_dates of the lifecycle          │      │  batch sessions (1..9),   │
  │                                               │      │  file link (file_ref)     │
  │  movements[] — TYPED MOVEMENTS, ordered:      │◀────▶│                           │
  │   { type: AUTHORIZATION|CAPTURE|CLEARING|     │ link │  postulate 1↔1 per tx,    │
  │     CHARGEBACK|FRAUD_ALERT,                   │ via  │  cardinality to be        │
  │     movement_id, correlation_id,              │ keys │  re-examined (variants    │
  │     business_time, amounts, status,           │      │  anticipated)             │
  │     remittance_ref?, scheme, _lineage }       │      └──────────────────────────┘
  │                                               │
  │  _lineage: source_system(s), refs, offsets    │      Chargeback & fraud =
  └───────────────────────────────────────────────┘      lifecycle movements,
                                                          attached via correlation_id
```

- **The transaction is an aggregate of typed movements**, not a status-bearing state: history is native, retroactive corrections (a late chargeback) are movements added, never overwrites.
- **The remittance is an independent object**, linked via stamping references — never embedded. Its own characteristics (batch sessions, gross debit/credit amounts, file link) live with it; the possible enrichment of remittance fields at transaction level is a logical-model decision, not a conceptual one.
- **Correlation keys are first-class**: indexed, mandatory, documented field by field (which key links what, with which cardinality).
- **Model versioning** (not stream versioning) is tooled: `schema_version` in every document, published changelog, model visualization and field dictionary (meaning, allowed values) maintained alongside the schema — not in orphaned pages.

### 2.4 v1 → v2 Migration Strategy Without Service Disruption

Two strategies are viable; the choice depends on the nature of the delta between v1 and v2.

#### Option A — Incremental In-Place Evolution (expand → migrate → contract)

Progressively mutate the v1 schema until it becomes v2, within the same collections. MongoDB lends itself to this (documents versioned via `schema_version`, readers tolerant to both shapes):

1. **Expand** — add the target structures (typed movements, remittance references) alongside the existing ones; writes feed both shapes.
2. **Migrate** — backfill historical documents to the target shape, in batches, as a background task.
3. **Contract** — remove v1 fields and behaviors once all readers have migrated.

**Strengths**: no second database, no API cutover, zero infrastructure cost. **Limits**: every intermediate step must remain readable by the v1 API → the read code handles N simultaneous shapes; when the delta is *structural* (externalizing the remittance, retyping movements, abandoning statuses), the trajectory becomes long and fragile, and the risk of stalling mid-way is real — the permanent intermediate state is the worst of all states.

#### Option B — Parallel v2, Dual Feeding, v2 API, Progressive Consumer Migration

```
        KAFKA (ingestion topics)                      ICEBERG RAW LAKE
             │            │                                 │
     consumer group   consumer group                        │  historical backfill
     organized-v1     organized-v2  ◀──── re-projection ────┘  (Spark/Flink batch,
             │            │               from raw             same mappings)
             ▼            ▼
     ┌──────────────┐  ┌──────────────┐
     │  v1 DATABASE │  │  v2 DATABASE │   dual feeding on the live stream
     │  (unchanged) │  │  (target)    │   (two independent consumer groups)
     └──────┬───────┘  └──────┬───────┘
            │                 │
        v1 API            v2 API          non-breaking contract: v1 frozen,
            │                 │           v2 documented, compat matrix published
     consumers   ──────────▶  progressive migration, consumer by consumer
            │                 │
            ▼                 ▼
     CONTINUOUS v1 ↔ v2 RECONCILIATION: counts per day/type/source,
     aggregate checksums (amounts), document sampling
            │
            ▼
     cutover criteria met → v1 freeze → decommissioning
```

1. **Create the v2 database** with the clean target model (§2.3) — free of inherited compromises.
2. **Feed v2 by re-projection from raw** for history (Iceberg backfill → v2 mappings → MongoDB) and via a **dedicated consumer group** for the live stream. Non-negotiable point: **no v1-database → v2-database ETL migration**, which would carry v1's defects and losses into v2. The re-projection also validates the platform's replayability promise along the way — if it fails, that is a major architectural signal to address.
3. **Dual feed v1 + v2** throughout the coexistence: two independent consumer groups on the same topics, no dependency between them.
4. **Expose the v2 API**; the v1 API is **frozen** (security fixes only) — the freeze is what makes the end-of-life credible.
5. **Migrate consumers progressively**, with continuous v1 ↔ v2 reconciliation as the proof of trust (the figures are identical, documented, defensible).
6. **Decommission** on explicit criteria: 100% of traffic on v2 for N weeks, reconciliation green, rollback plan expired.

> ✅ **Architect's recommendation**
>
> The v1 → v2 delta is **structural** (typed movements, externalized remittance, abandonment of statuses, tri-source K2/NEPTING/E-Stream convergence): **Option B is recommended**, with v2 fed by re-projection from raw. Option A remains the standard tool for later *additive* evolutions of v2 (new fields, new movement types) — the combination of the two constitutes the complete strategy: **B to change generations, A to live within a generation**. Common success condition: the raw pipeline of §1.2 must be in production *before* the migration begins, since it is its engine.

---

## 3. Summary of Architecture Principles

| # | Principle | Operational translation |
|---|---|---|
| 1 | **Contract before stream** | No producer onboarded without a versioned Avro contract, mandatory correlation keys, compatibility CI, golden samples. |
| 2 | **Two semantics, one mapping** | Contract = producer semantics; organized model = business semantics; versioned, tested mapping between the two. No leakage. |
| 3 | **Immutable, replayable raw** | Iceberg/Parquet, one table per event type, `days(business_time)` partition, 2–5 min commits, scheduled maintenance. |
| 4 | **The Kafka key is an architectural decision** | Key = correlation identity (high cardinality, per-aggregate ordering); 64–128 partitions from creation; never repartition live. |
| 5 | **Correlation at ingestion, never during processing** | Events carry their keys (contract); blind idempotent upserts; zero lookups in the hot path. |
| 6 | **Orphans are visible and attributable** | Batched asynchronous consolidation; per-producer metrics; notified non-correlation report; nothing disappears. |
| 7 | **The organized model describes the business** | Transaction = aggregate of typed movements; remittance = independent object; K2/NEPTING/E-Stream convergence proven by cross golden tests. |
| 8 | **Change generations by re-projection** | v2 fed from raw (never via v1→v2 ETL); dual feeding; v1 API frozen; continuous reconciliation; criteria-based decommissioning. |
| 9 | **The schema is versioned, not the streams** | `schema_version` in every document, published changelog, field dictionary and visualization maintained with the model. |

---

*Scope of this study: ingestion and organized model. The **exposition** (hot MongoDB APIs, KPI service) and **large-volume report extraction** (Starburst/Trino federating Iceberg + MongoDB, asynchronous exports) areas are covered in dedicated studies building on the foundations laid here.*
