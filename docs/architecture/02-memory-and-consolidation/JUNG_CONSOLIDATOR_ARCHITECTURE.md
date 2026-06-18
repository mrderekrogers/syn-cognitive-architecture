# Jung Consolidator

Syn's memory ingestion, storage, and consolidation service. Owns the
PostgreSQL schema for `memory_entries` and `memory_clusters`, writes
every memory to both Postgres AND filesystem backup, runs spectral
clustering on accumulated batches every hour, and exposes the query
surface (high-arousal, per-user, OH YEAH candidates, kicker pool)
that the rest of the system reads from.

This is the writer side of the memory system that `MEMORY_RECALL_AND_SALIENCE_ARCHITECTURE.md`
covers from the state-and-events side. Jung owns the schema;
`memory_salience` does the math; `memory_recall` updates state; DMN
orchestrates the cycle. This doc covers Jung.

Single module: `services/jung/consolidator.py` (~1050 lines).
FastAPI app with one background task (consolidation loop), one bus
subscriber (`CH_FREUD_PROFILE` → `handle_profile`), four HTTP
endpoints, and two Postgres tables plus their indices.

---

## Purpose and Design

The module comment is terse:

> Jung — Memory Consolidation via Spectral Clustering.
> Stores memory entries and clustering results in PostgreSQL.
> Indexes memories in Weaviate for semantic retrieval (RAG pipeline).
> Falls back to filesystem-only if PostgreSQL is unavailable.

Unpacked, Jung has four distinct responsibilities:

1. **Schema ownership.** `memory_entries` and `memory_clusters`
   tables are created and migrated here. No other module manages
   these schemas.
2. **Ingestion.** Memories arrive via the `CH_FREUD_PROFILE` bus
   channel (produced by Freud's user profiler). Each memory is
   inserted into Postgres, backed up to the filesystem, embedded
   via `llm_client.embed_text`, and indexed in Weaviate for RAG
   retrieval.
3. **Consolidation.** Every hour, pending memories are spectral-
   clustered (cosine-similarity affinity, precomputed matrix)
   into 2-10 clusters. Cluster IDs are written back to each
   memory's Postgres row.
4. **Query surface.** Other parts of the system read memories via
   named query functions — `pg_query_high_arousal` for dream seeds,
   `query_kicker_candidates` for the dream kicker, `pg_query_by_user`
   for user-context retrieval, `query_oh_yeah_candidates` for the
   OH YEAH semantic-match scan.

Graceful degradation is baked in. If Postgres is unavailable, `_get_pg`
returns `None` and every query returns `[]`; ingestion still writes
the filesystem backup. If `embed_text` fails, the memory is stored
without an embedding and gets handled via lexical fallback at
consolidation time.

---

## PostgreSQL Schema

### `memory_entries` — the primary table

```sql
CREATE TABLE memory_entries (
    id                   SERIAL PRIMARY KEY,
    content              TEXT NOT NULL,
    user_id              VARCHAR(128) NOT NULL DEFAULT '',
    vad_arousal          FLOAT DEFAULT 0.5,
    vad_valence          FLOAT DEFAULT 0.5,
    vad_dominance        FLOAT DEFAULT 0.5,
    embedding            JSONB,
    cluster_id           INTEGER,
    source               VARCHAR(64) DEFAULT 'freud',
    filename             VARCHAR(256),
    created_at           TIMESTAMP DEFAULT NOW(),

    -- Salience / activation fields
    vad_depth_cached     FLOAT,
    recall_count         INTEGER DEFAULT 0,
    last_recalled_ts     TIMESTAMP,
    activation_state     VARCHAR(16) DEFAULT 'active',
    salience_cached      FLOAT,
    salience_updated_at  TIMESTAMP
);
```

Owner: this module. Written by:
- `_pg_insert_memory` (Jung, on ingest): sets all fields except the
  `recall_count`, `last_recalled_ts`, `salience_cached`,
  `salience_updated_at` which start NULL/default.
- `_pg_update_clusters` (Jung, on consolidation): sets `cluster_id`.
- `memory_recall.record_retrieval` (external): bumps
  `recall_count`, sets `last_recalled_ts`.
- `memory_recall.handle_oh_yeah` (external): bumps `recall_count`
  by 5, sets `last_recalled_ts`, sets `activation_state = 'active'`.
- `memory_recall.recompute_all_salience` (external): sets
  `salience_cached`, `activation_state`, `salience_updated_at`.

### `memory_clusters` — batch telemetry

```sql
CREATE TABLE memory_clusters (
    id             SERIAL PRIMARY KEY,
    cluster_label  INTEGER NOT NULL,
    member_count   INTEGER DEFAULT 0,
    batch_id       VARCHAR(64),
    created_at     TIMESTAMP DEFAULT NOW()
);
```

**This table is batch telemetry, NOT semantic memory.** `cluster_label`
is batch-local and unstable — label `3` in batch A has no relationship
to label `3` in batch B. The durable cluster reference is
`memory_entries.cluster_id`.

Written only by `_pg_update_clusters` (one row per cluster per batch).
Pruned by `memory_recall.prune_memory_clusters` (daily, keep last
1000 batches).

(`_ensure_tables` runs an idempotent `ALTER TABLE memory_clusters
DROP COLUMN IF EXISTS centroid, theme` on every startup so deployments
that created the table under earlier schemas — which declared
`centroid JSONB` and `theme TEXT` columns that no writer ever
populated — get cleaned up automatically. If centroid (mean
embedding per cluster) or theme (LLM-generated cluster label) are
wanted later, the columns can be re-added as part of the design
pass that produces the value.)

### Indices

```sql
CREATE INDEX idx_memory_user         ON memory_entries(user_id);
CREATE INDEX idx_memory_cluster      ON memory_entries(cluster_id);
CREATE INDEX idx_memory_arousal      ON memory_entries(vad_arousal DESC);
CREATE INDEX idx_memory_activation   ON memory_entries(activation_state);
CREATE INDEX idx_memory_clusters_created ON memory_clusters(created_at DESC);
```

The activation-state index is the hot path for retrieval queries —
"ACTIVE-only" and "DORMANT-reachable" scans both filter on this
column. The created-at index on `memory_clusters` supports the daily
telemetry pruning scan efficiently.

### Migration strategy

`_ensure_tables()` does two things on startup:

1. `CREATE TABLE IF NOT EXISTS` for both tables — creates fresh
   instances with the full current schema.
2. `ALTER TABLE ... ADD COLUMN IF NOT EXISTS` for each column that
   might be missing on an older deployment. Specifically:
   `vad_valence`, `vad_dominance`, `vad_depth_cached`, `recall_count`,
   `last_recalled_ts`, `activation_state`, `salience_cached`,
   `salience_updated_at`.

Both are idempotent — safe to run on every service startup. The
`ALTER` approach means the module can ship schema changes in
additive form without a separate migration step. Column drops or
renames would still require manual migration.

---

## Module Map

### Schema management

| Function | Purpose |
|---|---|
| `_get_pg_dsn()` | Build the Postgres connection string. Reads `cluster.node1.host` config; port 5432, dbname `syn_memory`, user `syn`. Password comes from the `POSTGRES_PASSWORD` env var only — if unset, raises `RuntimeError` (caught by `_get_pg`, which then degrades to filesystem-only mode). No source default. |
| `_get_pg()` | Lazy pool initialization. 1-5 connections (ThreadedConnectionPool). Returns `None` on any failure — callers handle None. Calls `_ensure_tables()` on first successful connect. |
| `_ensure_tables()` | Idempotent `CREATE TABLE IF NOT EXISTS` + backward-compat `ALTER TABLE ADD COLUMN IF NOT EXISTS`. Runs once on pool creation. |

The pool is module-level singleton (`_pg_pool`); first caller pays
the initialization cost, subsequent callers use the cached pool.

### Ingest path

| Function | Signature | Purpose |
|---|---|---|
| `_pg_insert_memory` | `(content, user_id, vad_arousal, embedding, filename="", vad_valence=0.5, vad_dominance=0.5)` → `Optional[int]` | Insert a row; compute `vad_depth_cached` at insert time via `shared.vad_math.vad_archival_depth(VADVector(...))`. Returns `id` or None. The archival-depth geometry uses dead-zone arousal ([0.25, 0.5]) and symmetric distance on valence/dominance — a memory at canonical rest scores 0.0. See §S in `DESIGN_DECISIONS.md` for the rationale. |
| `JungConsolidator.ingest` | `async (content, user_id, vad_arousal=0.5, vad_valence=0.5, vad_dominance=0.5)` → `None` | Full ingest path: embed, Postgres insert (with full VAD triple), filesystem backup, Weaviate index, queue for clustering. |
| `handle_profile` | `async (msg: SynOutbound)` → `None` | Bus handler for `CH_FREUD_PROFILE`. Parses Freud's JSON payload, extracts text fields (`traits`, `topics`, `observations`, `emotional_dynamics`, `relational_shifts`, `trust_signals`), joins with ` \| ` separator, calls `jung.ingest`. |

**VAD-depth computation (via `shared/vad_math.vad_archival_depth`):**

```python
# shared/vad_math.py
def vad_archival_depth(vad: VADVector) -> float:
    dv = vad.valence - 0.5
    da_contrib = _arousal_archival_contribution(vad.arousal)
    dd = vad.dominance - 0.5
    distance = math.sqrt(dv * dv + da_contrib * da_contrib + dd * dd)
    return min(distance / _ARCHIVAL_MAX_DISTANCE, 1.0)

def _arousal_archival_contribution(arousal: float) -> float:
    # Dead zone: arousal in [0.25, 0.5] contributes 0 — the mundane
    # life-happening-at-normal-pace range is uninformative on its own.
    if 0.25 <= arousal <= 0.5:
        return 0.0
    if arousal < 0.25:
        return 0.25 - arousal        # suppression (exhaustion, numbness)
    return arousal - 0.5             # elevation (engagement, stress)
```

Range [0, 1]. Fresh at ingest, stored in `vad_depth_cached`, never
recomputed. The doctrine from `_pg_insert_memory`'s docstring:

> vad_depth is the memory's intrinsic weight — captured at formation,
> not re-evaluated later, because the state AT THE MOMENT OF
> FORMATION is what encodes how much that memory deserves to persist.

This is load-bearing for salience (`shared/memory_salience.py`) — the
time-decay half-life scales with `vad_depth`, so a weighty memory
decays slower. The full VAD triple flows from Freud (`profile_data["vad"]`
on `CH_FREUD_PROFILE`) through `handle_profile` → `jung.ingest` →
`_pg_insert_memory`, so `vad_depth_cached` reflects three-axis
geometry, not arousal-only.

### Query surface

All four query functions follow the same shape: lazy pool, thread-
offloaded `_query()` function, graceful empty-list return on failure.

| Function | Purpose | Records retrieval? |
|---|---|---|
| `pg_query_high_arousal(limit=20)` | Dream seeds. Filters `vad_arousal > 0.5` AND `activation_state IN ('active', 'dormant')`. Sorts ACTIVE-first, then by arousal DESC, created_at DESC. | Yes — fires `record_retrieval(id, SOURCE_DREAM_SEED)` for every seed returned, fire-and-forget via asyncio.create_task. |
| `query_kicker_candidates(limit=500)` | Dream kicker pool. Scans ALL activation states (including FADED) where `vad_depth_cached IS NOT NULL AND > 0`. Pre-sorts by `vad_depth_cached DESC, created_at DESC`. Caller uses `memory_salience.sample_kicker` for weighted draw. | No — the caller handles bumping if the kicker lands. |
| `pg_query_by_user(user_id, limit=50)` | User context retrieval. Filters to ACTIVE-only (dormant explicitly excluded — "routine user-context should reflect what she's actively carrying, not her whole history with someone"). | Yes — `record_retrieval(id, SOURCE_USER_CONTEXT)` per returned row, fire-and-forget. |
| `query_oh_yeah_candidates(activation_states=("dormant", "faded"), limit=500)` | OH YEAH scan candidate pool. Returns rows with embeddings for caller to compute cosine similarity. | No — caller decides which matches fire OH YEAH and handles the bump there. |
| `pg_get_stats()` | Simple count/average aggregates for the `/status` endpoint. | No. |

**Why some queries bump recall and others don't:**

- `pg_query_high_arousal` bumps because every dream seed IS a rehearsal;
  that rehearsal should count toward the memory's persistence.
- `pg_query_by_user` bumps because user-context retrieval IS ongoing
  engagement with that memory; it's the primary rehearsal signal.
- `query_kicker_candidates` doesn't bump at query time because the
  caller draws ONE kicker from the pool; bumping the other 499 would
  be noise. The caller handles the single-kicker bump through
  `record_retrieval` OR `handle_oh_yeah` depending on the prior
  state.
- `query_oh_yeah_candidates` doesn't bump because not every returned
  candidate becomes a match — only those crossing the 0.95 cosine
  threshold fire OH YEAH. The caller handles that downstream.

### Consolidation path

```python
class JungConsolidator:
    def __init__(self):
        self.pending_memories: list[MemoryEntry] = []
        self.consolidation_interval = 3600   # 60 min
        ...

    async def ingest(self, content, user_id, vad_arousal=0.3,
                     vad_valence=0.5, vad_dominance=0.5):
        ...
        self.pending_memories.append(entry)
        ...

    async def consolidate(self):
        ...
```

The consolidator holds a per-process in-memory queue of pending
`MemoryEntry` objects. On every `ingest`, the new memory is appended.
On every hour (or on `/consolidate` force-trigger), the queue is
drained and clustered.

**Consolidation flow:**

1. **Snapshot.** `to_process = self.pending_memories[:]; self.pending_memories = []`.
   Guards against races with concurrent ingests (comment marks this
   as "RACE FIX").
2. **Floor check.** If `len(embedded_memories) < 5`, put everything
   back and return. Spectral clustering needs at least 5 vectors to
   be meaningful.
3. **Partition.** Split `to_process` into `embedded_mems` (has
   embedding) and `unembedded_mems` (embedding call failed at ingest).
4. **Cluster the embedded batch.** Compute cosine similarity matrix,
   shift to [0, 1] for non-negative affinity, run `SpectralClustering`
   with `n_clusters = min(max(2, N // 5), 10)`.
5. **Assign cluster IDs** to each embedded memory.
6. **Lexical fallback for unembedded memories.** Tokenize content
   (whitespace-split, lowercase, >3 chars). Find the cluster with the
   most token overlap. Assign.
7. **Persist.** `_pg_update_clusters(batch_id, entries_with_embeddings + unembedded_mems)`.
   The function reads `cluster_id` directly off each entry, so passing
   embedded and lexical-fallback entries together unifies the persistence
   path — both kinds carry their cluster assignment into Postgres.
8. **Filesystem backup** of cluster summaries (JSON per cluster).
9. **Bus publish** on `CH_JUNG_CONSOLIDATION` with a summary message.

**On failure**, `to_process` is re-extended back into
`pending_memories` for next-cycle retry. This is mostly the right
behavior — a sklearn ImportError or transient failure shouldn't lose
memories — but the re-extend order is `to_process + self.pending_memories`,
so newer-ingested memories end up AFTER the retry batch, which is
fine (FIFO through the queue).

### Bus handlers

| Channel | Handler | Purpose |
|---|---|---|
| `CH_FREUD_PROFILE` | `handle_profile` | Freud's user profiles come in; text extracted + passed to `jung.ingest`. |

This is the **only** inbound channel to Jung. No other service
publishes to a channel Jung subscribes to. In practice,
`handle_profile` is the sole entry point for memories into the
system.

---

## The Fallback Strategy

Jung is designed to degrade gracefully on three axes:

### Postgres unavailable

`_get_pg` returns `None`. Every query function returns `[]`. Ingest
still writes the filesystem backup (`/data/syn/memories/{filename}.json`).

Consequence for the rest of the system: `memory_recall` functions
all return their "unavailable" defaults; retrieval paths still function
but surface fewer memories; the next time Postgres comes back, new
memories flow in cleanly (filesystem-only memories from the outage
are not auto-imported — they'd have to be backfilled manually).

### Embedding unavailable

If `embed_text` raises (transient inference failure or an upstream
outage), `entry.embedding = []`. The memory is inserted with
`embedding = NULL` in Postgres. Downstream:

- `query_oh_yeah_candidates` filters on `embedding IS NOT NULL` —
  the memory is excluded from OH YEAH scans. Can't semantically match
  without an embedding.
- `query_kicker_candidates` filters on `vad_depth_cached > 0` —
  independent of embedding, so the memory IS kicker-eligible.
- `pg_query_high_arousal` and `pg_query_by_user` don't look at
  `embedding` — memory is retrievable via these paths normally.
- Consolidation: `consolidate()` partitions the batch and runs the
  lexical fallback for un-embedded memories. See next section for the
  bug there.

### sklearn unavailable

Import fails. Consolidation is skipped; `to_process` is re-extended
back into `pending_memories`; the next cycle tries again. Memories
stay in the pending queue (and in Postgres, since `ingest` already
inserted them — they just have `cluster_id = NULL`).

### Weaviate unavailable

`shared.rag.index_memory` fails. Logged at debug; memory is still
inserted into Postgres and filesystem. RAG retrieval won't find the
memory semantically until it's backfilled, but Postgres-based
queries still work.

---

## Integration Points

### Inbound (who writes memories into Jung)

| Source | Channel / method | What triggers it |
|---|---|---|
| `services/freud/profiler.py` | Publishes `CH_FREUD_PROFILE` on every user profile update | Freud generates a user profile every N interactions; publishes the profile JSON for Jung + other subscribers |

That's the only inbound channel. No other service ingests memories
via Jung.

### Outbound (who reads from Jung)

| Consumer | What it calls |
|---|---|
| `shared/memory_recall.py` (`_get_pg` import path) | Pool access via `from services.jung.consolidator import _get_pg`. Used by `record_retrieval`, `handle_oh_yeah`, `recompute_all_salience`, `prune_memory_clusters`. |
| `shared/memory_recall.py::scan_conversational_oh_yeah` | `query_oh_yeah_candidates` |
| `services/dmn/scheduler.py::_gather_dream_seeds` | `pg_query_high_arousal` |
| `services/dmn/scheduler.py` dream kicker block inside `_gather_dream_seeds` | `query_kicker_candidates` |
| Various user-context pulls in retrieval pipeline | `pg_query_by_user` |
| `/status` HTTP endpoint | `pg_get_stats` |
| `memory_recall.handle_oh_yeah`'s diary-entry path | Reads Postgres via `_get_pg` indirectly |

`shared/memory_recall` has the tightest coupling — it uses Jung's
pool accessor directly rather than going through a query function.
This is the convention for "runtime state updater" vs "schema owner":
Jung owns the schema, `memory_recall` needs direct-SQL access to do
batch updates efficiently.

### Outbound (what Jung produces)

- Bus events on `CH_JUNG_CONSOLIDATION` (once per consolidation
  batch).
- Postgres rows in `memory_entries` and `memory_clusters`.
- Filesystem files in `/data/syn/memories/`:
  - `mem_{timestamp}_{user_id}.json` — one per memory
  - `cluster_{timestamp}_{cluster_id}.json` — one per cluster per batch
- Weaviate index entries (via `shared.rag.index_memory`).

---

## FastAPI Surface

### `GET /status`

```json
{
  "pending_memories": 12,
  "last_consolidation": 1712345678.0,
  "total_ingested": 458,
  "total_consolidated": 430,
  "postgresql": {
    "status": "connected",
    "total_entries": 458,
    "total_clusters": 24,
    "unique_users": 6,
    "avg_arousal": 0.412
  }
}
```

### `POST /consolidate`

Force-trigger a consolidation cycle. Useful for testing; normally
runs on the 60-minute cadence automatically.

### `GET /high_arousal?limit=10`

Wrapper around `pg_query_high_arousal`. Operator-facing.

### `GET /user/{user_id}?limit=20`

Wrapper around `pg_query_by_user`. Operator-facing.

Plus the standard `/health` endpoint added via `shared.health.add_health_endpoint(app)`.

---

## Invariants and Contracts

1. **Jung owns the `memory_entries` and `memory_clusters` schemas.**
   Other modules read and update rows via direct SQL, but schema
   changes (new columns, new indices) land here.
2. **`vad_depth_cached` is computed at ingest and never updated.**
   Captures "emotional weight at moment of formation." Don't add a
   recomputation path without understanding that the salience model
   reads this as a stable intrinsic property.
3. **Ingest always writes filesystem backup, even when Postgres
   succeeds.** Filesystem is the last-resort recovery surface if
   Postgres is corrupted.
4. **`activation_state` defaults to `'active'` on insert.** New
   memories are immediately retrieval-eligible, even before the first
   nightly salience recompute.
5. **`cluster_id` defaults to `NULL`** and is only set on
   consolidation. A memory in `pending_memories` awaiting
   consolidation has cluster_id=NULL in Postgres.
6. **Consolidation snapshot-then-reset is race-safe.** Don't remove
   the "RACE FIX" pattern at the top of `JungConsolidator.consolidate()`
   (snapshot the pending list with `to_process = self.pending_memories[:]`,
   then reset `self.pending_memories = []` BEFORE the first await) —
   it guards against concurrent ingest during consolidation.
7. **Embedding failures don't block ingest.** Memories with
   embedding=NULL are valid and persist; they're only excluded from
   embedding-dependent paths (OH YEAH scan).
8. **Postgres unavailability is non-fatal everywhere.** Every
   `_get_pg` caller handles None. A Postgres outage shouldn't
   propagate anywhere except to degraded retrieval.
9. **Bus publication on `CH_JUNG_CONSOLIDATION` happens AFTER DB
   writes commit.** Don't reorder. Subscribers assume the DB state
   reflects the batch by the time they see the event.

---

## Related Docs

- `MEMORY_RECALL_AND_SALIENCE_ARCHITECTURE.md` — the companion doc.
  Jung owns the schema; `memory_salience` does the math;
  `memory_recall` updates state. Read together for the full memory
  picture.
- `MEMORY_ARCHITECTURE.md` — the conceptual parent. Memory types,
  file format, dream distillation, the overall philosophy.
- `LLM_CLIENT_ARCHITECTURE.md` — `embed_text` dependency.
- `services/freud/profiler.py` (not currently documented) — the
  upstream producer of memories. A future `FREUD_ARCHITECTURE.md`
  would cover user profiling, interaction tracking, and the VAD
  attachment to `CH_FREUD_PROFILE` payloads that feeds Jung's
  three-axis archival depth.
- `shared/rag.py` (not currently documented) — the Weaviate indexing
  layer called from `jung.ingest`. Currently best-effort with
  graceful failure.
- `DESIGN_DECISIONS.md` §P *VAD Consolidation* — the `VADVector`
  used for in-memory VAD handling. The `vad_depth_cached` schema
  column implements the persistent side of the same concept.
