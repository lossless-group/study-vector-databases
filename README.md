# Vector Databases — Study

## The question

> How do open-source vector / context databases structure embedding
> storage, recall ranking, payload schemas, filter expressivity,
> persistence, and ANN-algorithm choice (HNSW / IVF / DiskANN /
> ScaNN / etc.) — and what are the trade-offs when migrating between
> them?

The interesting bits are **not** "which one is fastest on which
benchmark." Vendor benchmarks are noisy and the leaderboard rotates
every quarter. The interesting bits are:

- **Payload model.** What can each record carry alongside the vector?
  Typed scalars, JSON blobs, references to other records, full
  documents? Does the DB enforce a schema or accept anything?
- **Filter expressivity.** Can you filter by `eq / ne / gt / in /
  contains`? Boolean composition (`AND / OR / NOT`)? Filter *during*
  ANN search (push-down) or *after* (post-filter)? How badly does
  filtering degrade recall?
- **Index algorithm and trade-offs.** HNSW (memory-resident, fast
  recall), IVF (disk-friendly, lossier), DiskANN / ScaNN (large-scale,
  hybrid). Which does each system default to, which can be tuned, and
  what's the build-time vs. query-time vs. memory-footprint trade-off?
- **Hybrid retrieval.** First-class BM25 / sparse vectors / keyword?
  Reciprocal Rank Fusion built-in? Or pure dense and you bring the
  fusion?
- **Scoping & multi-tenancy.** Collections, namespaces, partitions,
  per-tenant indexes — how does each model isolate data, and what does
  it cost?
- **Persistence & operations.** In-process, embedded server, networked
  server, distributed? What's the durability story? Backup / restore /
  point-in-time? Live re-index?
- **Serialization on disk.** What does the persisted form look like?
  Could a non-AI program parse it cleanly without the DB binary?
- **Migration cost.** What does it take to move 10M vectors from one to
  another? Are the embedding metadata and IDs portable?

## Why this study exists

[memory-layers-for-agents](../memory-layers-for-agents) studies the
**opinionated layer**: Mem0, Letta, Honcho, RetainDB, etc. — systems
that decide what to extract, when to write, how to scope, how to rank.
Those systems all sit on top of a **primitive** — a vector / context
database — that has none of those opinions and a different design
surface entirely.

The Lossless Group currently runs **Chroma** in production
(`ai-labs/context-vigilance-kit/` ingests `context-v/`, `changelog/`,
Claude session traces, and tool traces into four local Chroma
collections, surfaced via the `chroma` MCP server). We have no lived
experience with anything else. This study is the prep work for the
question *"if we ever needed to move off Chroma — or run Chroma plus
something else — what would we be giving up and what would we be
gaining?"*

## What we are looking at, repo by repo

When reading each entry below, the working checklist is the bullet
list above — payload, filters, indexes, hybrid, scoping, persistence,
serialization, migration cost.

---

## The design space at a glance

| Bet | Entry |
|---|---|
| Embedded-first, "context DB" framing, batteries-included | [Chroma](./chroma) |
| Rust + payload-first + rich filtering + strong defaults | [Qdrant](./qdrant) |
| GraphQL + modules + schema-on-write + RAG-friendly | [Weaviate](./weaviate) |
| Distributed by design, multiple ANN backends, Kubernetes-scale | [Milvus](./milvus) |
| Columnar Arrow on disk + in-process + structured-data co-location | [LanceDB](./lancedb) |
| Postgres extension; vectors as one column type alongside everything else | [pgvector](./pgvector) |

Deep per-entry write-ups live in [`context-v/profiles/`](./context-v/profiles).

## In the study now

### [chroma](./chroma)
- **Repo:** https://github.com/chroma-core/chroma — *the AI-native open-source embedding database*
- **Maintainer:** Chroma (`chroma-core` org); Apache-2.0
- **Why this is here:** The **baseline reference** — what the Lossless
  Group runs today. Chroma's framing is "context DB for AI," not
  "vector DB" — distinct positioning compared to Qdrant / Weaviate /
  Milvus, which lead with the ANN story. Embedded by default
  (Python `EphemeralClient`, persistent `PersistentClient`), with a
  separate server (`chroma run`) and a hosted tier (Chroma Cloud)
  for scale-out. The headline design choices: collection-as-namespace,
  document + embedding + metadata as the unit, where-filter language
  with `$eq / $ne / $gt / $in / $and / $or / $not`, and the more
  recent **Schema + Search** (hybrid retrieval) primitives shipping in
  the Cloud tier. The interesting comparison is against LanceDB
  (which also bets on embedded) and pgvector (which also bets on
  "your other data lives here too").

### [qdrant](./qdrant)
- **Repo:** https://github.com/qdrant/qdrant — *high-performance, massive-scale vector database*
- **Maintainer:** Qdrant (`qdrant` org); Apache-2.0
- **Why this is here:** Mem0's default backend (per
  `studies/memory-layers-for-agents/context-v/profiles/Profile__Mem0.md`)
  and the most direct "migrate off Chroma" candidate by API shape.
  Written in Rust with a payload-first design — every point carries a
  rich JSON payload that filters operate over with full predicate
  expressivity (geo, range, full-text indexes on payload fields), and
  filtering is *pushed into* the ANN traversal rather than applied
  after the fact. HNSW is the index, with quantization (scalar,
  binary, product) as first-class knobs. Sparse vectors and named
  multi-vector points are supported. Operational story: embedded
  (in-process via Python/Rust bindings), single-node server, and
  distributed cluster mode — same API across all three. The contrast
  against Chroma is *operational maturity vs. embedded ergonomics*.

### [weaviate](./weaviate)
- **Repo:** https://github.com/weaviate/weaviate — *cloud-native open-source vector database with GraphQL + REST*
- **Maintainer:** Weaviate (`weaviate` org); BSD-3-Clause
- **Why this is here:** The schema-on-write bet — classes,
  properties, types, references between objects, all enforced before
  storage. GraphQL as the primary query language is a distinctive
  choice that closer-mirrors how application code thinks about data
  than the "collection + filter dict" shape Chroma/Qdrant use.
  Modules (`text2vec-openai`, `text2vec-cohere`, `text2vec-huggingface`,
  reranker modules, generative modules) let the DB own the
  embedding-and-RAG pipeline rather than expecting the application to.
  The interesting comparison is against RetainDB (already in
  memory-layers-for-agents) — both bet that *typing* is the
  load-bearing primitive, but Weaviate types at the storage layer
  and RetainDB types at the memory-category layer above it.

### [milvus](./milvus)
- **Repo:** https://github.com/milvus-io/milvus — *cloud-native vector database for scalable similarity search*
- **Maintainer:** Zilliz (`milvus-io` org); Apache-2.0
- **Why this is here:** The Kubernetes-scale answer. Milvus is the
  only entry in the study that's distributed-by-default — separate
  compute and storage tiers, multiple ANN backends pluggable per
  collection (HNSW, IVF, DiskANN, ScaNN, GPU-accelerated variants),
  and a partitioning model designed for billions of vectors.
  Lite/Standalone modes exist for development but the design center
  is "you need a vector DB that survives k8s pod restarts and
  horizontal scale-out." Worth pinning as the "what would we get if
  we outgrew our current scale entirely" data point — even though
  it's probably overkill for the Lossless Group's current needs.

### [lancedb](./lancedb)
- **Repo:** https://github.com/lancedb/lancedb — *developer-friendly serverless vector database*
- **Maintainer:** LanceDB (`lancedb` org); Apache-2.0
- **Why this is here:** The columnar-on-disk bet. Storage is the
  **Lance format** (Apache Arrow-derived, columnar, zero-copy reads),
  which means vectors live alongside structured columns the way
  Parquet stores tabular data — no separate "vector store +
  metadata DB" split. Embedded-first like Chroma (Python, Node,
  Rust, Java SDKs all in-process), but the format is the
  distinguishing piece: a `.lance` directory is readable by any
  Arrow consumer (DuckDB, Polars, Pandas), which directly answers
  the "could a non-AI program parse this" checklist item that most
  vector DBs fail. The contrast against Chroma is *file-format
  portability vs. application ergonomics*; against pgvector,
  *purpose-built for vectors-plus-columns vs. retrofitted onto
  Postgres*.

### [pgvector](./pgvector)
- **Repo:** https://github.com/pgvector/pgvector — *open-source vector similarity search for Postgres*
- **Maintainer:** Andrew Kane (`ankane`); PostgreSQL License
- **Why this is here:** The "vectors are just another column type"
  bet. pgvector is a Postgres extension — once installed, you have
  `vector(N)` as a first-class type, `<->` / `<=>` / `<#>` operators
  for distance, HNSW and IVFFlat indexes, and **the full Postgres
  expression language for filtering**. The reason most of the
  memory-layer entries (Honcho, Letta, RetainDB, Hindsight, Volt all
  per the memory-layers-for-agents profiles) run on top of it: if
  you're already operating Postgres for your transactional data,
  the marginal cost of *also* doing vector search is one
  `CREATE EXTENSION` call. The contrast against everything else in
  this study: pgvector concedes the pure-ANN performance crown in
  exchange for inheriting Postgres's transactions, joins, backups,
  replication, ACL, and operational tooling unchanged.

---

## Candidates to consider adding later

These are worth knowing about but deliberately not pinned in the
initial scaffold. The study's center of gravity is "what could we
realistically migrate Chroma to or run alongside it" — adding more
would dilute that focus.

### Other OSS vector / hybrid stores

- **redis** with RediSearch / Vector — Apache-2.0 (core) / RSALv2.
  KV-first, vector secondary. https://github.com/redis/redis
- **vespa** — Apache-2.0. Hybrid search engine with vector + full-text
  + structured filters and a tensor language. Massive surface.
  https://github.com/vespa-engine/vespa
- **opensearch** — Apache-2.0. Elasticsearch fork with vector support
  via `knn_vector`. https://github.com/opensearch-project/OpenSearch
- **tantivy** — MIT. Full-text first, with vector being added; a
  Rust analogue to Lucene. https://github.com/quickwit-oss/tantivy

### Proprietary or restricted-license vector stores

These are not OSS but show up in the agent-memory ecosystem and are
worth referencing without pinning:

- **Turbopuffer** — proprietary, but Honcho ships an adapter
  (see `studies/memory-layers-for-agents/honcho/src/vector_store/`).
  Object-storage-backed; the architectural curiosity is "what if the
  vector index lived on S3."
- **Pinecone** — proprietary, fully managed.
- **Chroma Cloud** — the hosted tier of an entry already in the
  study; relevant when comparing the embedded-vs-hosted operational
  story.

---

## Reading order suggestion

1. Start with **Chroma**'s `chromadb/` Python package and the
   `chromadb/api/` directory — the API shape we already use is the
   baseline against which everything else is read.
2. Compare to **Qdrant**'s `rust/qdrant/src/` — same problem,
   different language, different payload posture.
3. Compare to **LanceDB**'s storage format spec (in the `lance`
   repo it depends on) — same problem, different *disk* posture.
4. Skim **pgvector**'s C source (`src/`) — it's small (~3k LoC) and
   reveals exactly how little a "vector DB" needs to be when you
   already have a relational engine.

By the time those four are read in sequence, the design space should
be mostly visible and Weaviate / Milvus / the candidates section
will read as "different operational answer to the same primitives."
