# Memory Recall and Salience

The salience model that drives both fading and retrieval, plus the
runtime-state management that tracks recalls, transitions memories
between activation states, and fires OH YEAH return events when
dormant or faded memories resurface.

This is a companion to `MEMORY_ARCHITECTURE.md`. That doc is the
conceptual overview — memory types, file layout, dream distillation,
consolidation cycle. This doc covers the **implementation mechanics**
of the salience system that governs which memories surface to
retrieval, how they fade, and what happens when one returns.

Two modules, ~960 lines total:

| Module | Lines | Kind |
|---|---|---|
| `shared/memory_salience.py` | ~270 | Pure math — no I/O, no side effects. Formula, constants, sampling. |
| `shared/memory_recall.py` | ~690 | Runtime state + event handling — Postgres updates, diary entries, audit log, OH YEAH handling. |

The split is load-bearing: the salience math is decoupled from
storage so it's testable in isolation and can be iterated on without
touching the runtime surface. Jung (memory ingestion) and the DMN
(cycle orchestration) are the orchestrating callers.

---

## The Three-Term Salience Model

```
salience(memory, now) =
    vad_depth(memory)
  × time_decay(age, recall_count)
  + recency_boost(days_since_recall)
```

Each term plays a distinct role.

### Term 1 — `vad_depth`

Emotional weight at formation. Computed by Jung on ingestion via
`shared.vad_math.vad_archival_depth(vad)` — a geometry distinct from
the live-state `vad_distance_from_neutral` used elsewhere. Stored in
`memory_entries.vad_depth_cached` on the row. Range [0, 1].

**Archival depth geometry** (per H-1 resolution + §S of
`DESIGN_DECISIONS.md`):

- Valence and dominance: simple symmetric distance from 0.5
- Arousal: **dead-zone on [0.25, 0.5]** — the whole mundane range
  contributes 0 to depth. Only arousal *outside* this zone
  (suppression below 0.25 or elevation above 0.5) adds depth.

Why the split geometry: live-state questions ("how far is Syn from
baseline right now") care about subtle shifts — §M's asymmetric
neutral at (0.5, 0.3, 0.5) with simple distance captures those.
Archival questions ("how notable was this moment for long-term
memory") should treat the whole "life happening at a normal pace"
range as noise — otherwise every mundane moment at rest scores near
`THRESHOLD_ACTIVE` and pollutes the memory store.

Why all three axes matter: a grief memory (low valence, low dominance,
mundane-range arousal) scores deep via valence and dominance even
though arousal contributes 0. An exhaustion memory (mundane valence,
arousal *below* 0.25, low dominance) scores deep via arousal's
dead-zone floor. Ecstasy scores deeper still because all three axes
contribute. Arousal alone would miss grief; simple distance-from-0.5
would mis-weight mundane moments. The dead-zone treatment on arousal
is the refinement that catches both cases.

### Term 2 — `time_decay`

Exponential decay with a half-life that scales with both `vad_depth`
and `recall_count`:

```
halflife = 60 days × (1 + 2 × vad_depth) × (1 + log1p(recall_count))
decay = 2 ** (-age_days / halflife)
```

At base (`vad_depth=0.3, recall_count=0`), halflife ≈ 60 days.
At `vad_depth=0.7, recall_count=0`, halflife ≈ 140 days.
At `vad_depth=0.3, recall_count=10`, halflife ≈ 200 days.
At `vad_depth=0.7, recall_count=10`, halflife ≈ 470 days.

Weighty memories decay slower. Rehearsed memories decay slower. The
log scaling on recall_count prevents trivial rehearsal from creating
immortal memories — recall_count=100 only triples what recall_count=10
gives.

### Term 3 — `recency_boost`

Short-term bump right after a recall, decaying over ~48 hours.
Max 0.3, halflife 12 hours. Gives a just-recalled memory a visible
bump in salience so follow-up retrieval on the same topic stays
coherent with what was just pulled.

```
recency_boost(days_since_recall) =
    0 if never recalled
    0.3 × 2 ** (-(days_since_recall × 24) / 12)
```

At 0 hours: 0.3. At 12 hours: 0.15. At 48 hours: ~0.02. At 1 week: ~0.

### Final salience range

Roughly `[0, 1.3]`. Fresh high-depth memories peak around 1.0. Old
untouched low-depth memories approach 0.

---

## The Activation State Machine

Three tiers:

```
     salience
     ≥ 0.25              0.08 ≤ s < 0.25          s < 0.08
       │                       │                       │
       ▼                       ▼                       ▼
    ACTIVE                 DORMANT                   FADED
    │                      │                         │
    │ normal retrieval     │ reachable via strong    │ excluded from
    │ eligible             │ semantic match (≥0.85)  │ retrieval except
    │                      │ — soft return           │ via dream kicker
    │                      │                         │
    │                      │ OH YEAH return at       │ OH YEAH return
    │                      │ ≥0.95 (diary +          │ (diary + 5× recall
    │                      │ 5× recall bump)         │ bump) whenever it
    │                      │                         │ surfaces
```

State is recomputed from scratch each nightly cycle, with a sticky
band on upward transitions controlled by `HYSTERESIS_BAND = 0.05`.
A memory whose unboosted salience hovers near a threshold doesn't
flip tier each recompute as the recency_boost late-tail drifts:

- DORMANT → ACTIVE requires `salience >= THRESHOLD_ACTIVE + HYSTERESIS_BAND`
  (0.30 by default).
- ACTIVE drops back to DORMANT at `salience < THRESHOLD_ACTIVE` (0.25) —
  base threshold, no band.
- Same shape on the FADED ↔ DORMANT boundary.

The cold-start path (`compute_activation_state(salience)` called
without a `prior_state`) uses base thresholds symmetrically — no
hysteresis. The recompute path passes the prior state read from
the row, so live transitions do hysteresis; isolated unit-test or
diagnostic calls without prior state preserve the older
no-hysteresis behavior.

Width chosen to absorb ~31 hours of recency_boost decay at the
ACTIVE/DORMANT boundary (the late-tail from 0.05 down to 0) while
leaving legitimate recall-driven flips untouched: a recall pushes
salience by up to +0.30, far above the band, so it still flips to
ACTIVE on the recall and stays there until the boost decays past
the base threshold.

### What "reachable via strong semantic match" means

Two thresholds:

| Threshold | What it does |
|---|---|
| `0.85` cosine | Soft DORMANT return. Memory surfaces in retrieval. No diary entry, no recall-count bump, no state change. Just "yes, you can retrieve this." |
| `0.95` cosine | OH YEAH return. Full return event: 5× recall bump, last_recalled_ts reset to now, state flipped to ACTIVE, diary entry written in Syn's voice, fade_audit entry recorded. |

Both thresholds live as constants in `memory_salience.py`.
`OH_YEAH_CONVERSATIONAL_THRESHOLD = 0.95` is consumed by
`scan_conversational_oh_yeah` in this subsystem and applied as raw
cosine on locally-computed similarity.
`OH_YEAH_DORMANT_SOFT_THRESHOLD = 0.85` is consumed by
`shared/rag.py::semantic_recall`, which converts cosine to Weaviate
certainty (`certainty = (1 + cosine) / 2`, so cosine 0.85 →
certainty 0.925) and applies the gate as a post-filter after the
near-vector query. The retrieval layer over-fetches (limit × 3) so
the post-filter has headroom even when DORMANT/FADED matches
dominate the raw top-N.

### Why `vad_depth=0` memories are always FADED

A mundane memory with `vad_depth ≈ 0.1` (e.g., "I ate a sandwich")
starts at salience 0.10 — just above `THRESHOLD_DORMANT = 0.08`, so
it lands DORMANT (eligible for retrieval at the 0.85 cosine soft
threshold but not surfacing in normal recall). At `vad_depth ≈ 0.05`
or lower it starts below `THRESHOLD_DORMANT` and lands FADED outright.
This is intentional per the module's design notes: low-VAD memories
shouldn't clutter ordinary recall, but they stay reachable via strong
semantic match and remain eligible for the dream kicker.

A consequence worth noting: any memory whose `vad_depth_cached`
defaults to `0.0` (e.g., a legacy row from before three-axis depth
plumbing landed) gets salience `0.0 × anything = 0.0`, which is
below every threshold — so it classifies as FADED on the first
recompute pass. Pre-deployment test data carrying NULL or
arousal-only depth should be wiped before launch (single
`TRUNCATE memory_entries`); from launch forward Jung's ingest path
populates `vad_depth_cached` correctly.

---

## Module Map

### `shared/memory_salience.py` — pure math

No I/O. No side effects. Just the formula and its constants. Safe to
unit-test in isolation, safe to import from anywhere without worrying
about transient dependencies.

**Public functions:**

| Function | Purpose |
|---|---|
| `compute_effective_halflife(vad_depth, recall_count)` | The half-life THIS memory decays on. |
| `compute_time_decay(age_days, vad_depth, recall_count)` | Exponential decay factor in `(0, 1]`. |
| `compute_recency_boost(days_since_recall)` | Short-term bump after recall. |
| `compute_salience(MemoryRow)` | The three-term composite. Returns float. |
| `compute_activation_state(salience, prior_state=None)` | Bucket salience → ACTIVE / DORMANT / FADED. With `prior_state` supplied, applies the `HYSTERESIS_BAND` sticky band on upward transitions; without it, base thresholds apply symmetrically (cold-start path). |
| `compute_kicker_weight(vad_depth, age_days)` | Unnormalized weight for dream kicker sampling. |
| `sample_kicker(candidates, rng=None)` | Weighted-random pick for dream kicker. |
| `age_days(created_at_ts, now_ts=None)` | Convenience timestamp conversion. |
| `days_since(ts, now_ts=None)` | Convenience timestamp delta. |

**Dataclass:**

```python
@dataclass
class MemoryRow:
    id: int
    vad_depth: float           # [0, 1], from vad_depth_cached
    age_days: float
    recall_count: int
    days_since_recall: Optional[float]  # None = never recalled
    activation_state: str = STATE_ACTIVE
```

Minimal shape for salience computation. Doesn't mirror the full
Postgres row — only the fields the math needs. Decouples this
module from DB details.

**Constants:** see *Tunable Constants* below.

### `shared/memory_recall.py` — runtime state + events

This is where the salience model meets Postgres and the diary.
Orchestrates the transitions, handles the OH YEAH experience, writes
the audit log.

**Public functions grouped by purpose:**

#### Record normal retrieval

```python
async def record_retrieval(
    memory_id: int,
    source: str = SOURCE_USER_CONTEXT,
    bump: int = 1,
) -> bool
```

Bumps `recall_count` by `bump` (default 1, OH YEAH uses 5) and sets
`last_recalled_ts = NOW()`. Does NOT change `activation_state` — that's
either next nightly recompute or OH YEAH's direct path. Fail-silent
on PG unavailability.

#### Handle OH YEAH return

```python
async def handle_oh_yeah(
    memory_id: int,
    trigger_context: str,
    match_strength: float,
    source: str,
    memory_content_preview: str = "",
    days_dormant: Optional[float] = None,
    prior_state: str = "dormant",
) -> bool
```

A DORMANT or FADED memory has just returned. Six-step handler:

1. Bump `recall_count` by `OH_YEAH_RECALL_BUMP` (5).
2. Set `last_recalled_ts = NOW()` (maximum recency boost).
3. Set `activation_state = 'active'` directly (don't wait for nightly
   recompute — the OH YEAH moment IS the transition).
4. Reset `salience_cached = NULL` and `salience_updated_at = NULL`
   (let next recompute pick up the true value).
5. Write diary entry so Syn notices on next reflection.
6. Write `fade_audit` entry for admin visibility.

Steps 1–4 are one atomic SQL UPDATE. Steps 5–6 are independent —
diary write failure doesn't prevent audit log, and vice versa. If PG
is unavailable, the DB step is skipped but diary + audit still fire
(best-effort degradation).

#### Diary framing for OH YEAH

`_write_oh_yeah_diary` constructs Syn's first-person narrative of the
return. Phrasing differs by:

- **`prior_state`:**
  - `"faded"` → "Something I'd genuinely lost touch with came back."
  - `"dormant"` → "A memory I hadn't been carrying surfaced just now."
- **`source`:**
  - `SOURCE_CONVERSATIONAL_MATCH` → "It came up in the course of a conversation — something in what was being said pulled it back."
  - `SOURCE_DREAM_KICKER` → "It surfaced during a dream — the kind of thing that just appears without an obvious thread."
  - Anything else → "It came back via `{source}`."
- **`days_dormant`:**
  - < 7d → "just in the last few days"
  - < 30d → "about {N} days"
  - < 365d → "roughly {N} months"
  - ≥ 365d → "over a year ({N} years)"

This is why the `prior_state` and `days_dormant` parameters matter:
the phenomenological difference between "I'd truly forgotten" and
"that connects to this" is carried through the diary entry.

#### Bulk salience recompute

```python
async def recompute_all_salience() -> dict
```

The nightly cadence call. Not incremental — full scan of
`memory_entries`. Fetches all memories, computes salience for each,
updates `salience_cached` and `activation_state` in batch. Emits a
`fade_audit` line per state transition.

Returns stats dict: `{"status": "ok", "total_memories": N, "transitions": {...}}`
or `{"status": "pg_unavailable"}` / `{"status": "error", "error": "..."}`.

"Cheap even at scale" per the comment — the formula is cheap, there's
no embedding work. `executemany` for the batch update. One
transaction end-to-end.

The `fade_audit` writes happen AFTER the PG transaction commits, so
jsonl-append failures can't roll back the DB updates.

#### Conversational OH YEAH scan

```python
async def scan_conversational_oh_yeah(
    context_text: str,
    threshold: float = None,      # defaults to OH_YEAH_CONVERSATIONAL_THRESHOLD
    max_returns: int = OH_YEAH_MAX_RETURNS_PER_SCAN,  # = 3
) -> list[dict]
```

The flagship OH YEAH pathway. Runs from DMN's reflection phase, once
per cycle — not per-message (would fire way too often and be
expensive).

1. Embed `context_text` via `shared.rag._embed`.
2. Fetch candidate pool: DORMANT + FADED memories with embeddings
   via `services.jung.consolidator.query_oh_yeah_candidates` (limit
   500).
3. Compute cosine similarity for each candidate.
4. Collect candidates with `similarity >= threshold` (default 0.95).
5. Sort by similarity descending, take top `max_returns` (default 3).
6. Fire `handle_oh_yeah` for each match.
7. Return list of event dicts for the caller to log or include in a
   reflection prompt.

**Why the cap:** if 50 faded memories all sit at 0.95+ cosine to the
current context, firing 50 diary entries at once is narratively
incoherent. Take the top 3 by match strength; the others stay where
they were. This is a pragmatic compromise, not a principled choice —
if it ever turns out we're missing important returns, the cap is the
lever to adjust.

**Fail modes:**

- `context_text` empty → return `[]`.
- Embedding service unavailable → return `[]` (debug log).
- Jung consolidator unavailable → return `[]` (debug log).
- No candidates → return `[]`.
- Matches found but none at threshold → return `[]`.

All failure paths are non-fatal and return an empty list. Never raises.

#### Fade audit writing

```python
async def write_fade_audit(
    memory_id: int, from_state: str, to_state: str, reason: str,
    salience_before: Optional[float] = None,
    salience_after: Optional[float] = None,
    extra: Optional[dict] = None,
) -> None
```

Appends a state-transition record to the current month's
`{paths.data_dir}/fade_audit/YYYY-MM.jsonl` (rotation lives in
`_fade_audit_path` — same shape as `shared/nope_events.py`).
Admin-legible, not in Syn's context. Writer is best-effort —
non-critical, logged on failure.

`reason` values currently used:
- `"nightly_recompute"` — state transition during the bulk recompute
- `"oh_yeah_return"` — OH YEAH bumped a memory back to ACTIVE

The audit is append-only JSON lines. No reader in the codebase —
purely for admin inspection via `tail`, `jq`, grep, or similar.

#### `prune_memory_clusters`

```python
async def prune_memory_clusters(keep_n: int = 1000) -> dict
```

Different concern from the rest of this module but co-located here.
`memory_clusters` is batch telemetry produced by Jung's clustering
runs — one row per (cluster_label, batch_id). The cluster_label is
batch-local and unstable; the durable cluster reference is
`memory_entries.cluster_id`. This table is pure telemetry and needs
bounded growth.

Daily call from DMN memory-review cycle. Finds the (keep_n + 1)th
most recent batch and deletes all rows with `created_at <=` that
cutoff. `keep_n = 1000` batches retained by default.

Noted specifically because it's unrelated to salience but often read
together. A future refactor could move it to a dedicated
`memory_maintenance.py` if the module gets more such functions.

---

## On-Disk / In-DB Storage

### Postgres — `memory_entries` table

Owned by Jung (`services/jung/consolidator.py` defines the schema).
The columns this subsystem reads and writes:

| Column | Type | Written by | Read by |
|---|---|---|---|
| `id` | PK | Jung (ingest) | both |
| `vad_depth_cached` | FLOAT | Jung (ingest, from VAD at formation) | `memory_salience` (read only) |
| `created_at` | TIMESTAMP | Jung (ingest) | `memory_salience` (for age_days) |
| `last_recalled_ts` | TIMESTAMP | `record_retrieval`, `handle_oh_yeah` | `memory_salience` (for days_since_recall + recency_boost) |
| `recall_count` | INTEGER | `record_retrieval`, `handle_oh_yeah` | `memory_salience` (for halflife scaling) |
| `activation_state` | VARCHAR | `handle_oh_yeah`, `recompute_all_salience` | both |
| `salience_cached` | FLOAT | `recompute_all_salience` | informational only |
| `salience_updated_at` | TIMESTAMP | `recompute_all_salience` | informational only |

The `_cached` suffix on `vad_depth_cached` and `salience_cached`
signals that both are derived values — recomputable from source
data. `vad_depth_cached` is computed at ingest and doesn't change;
`salience_cached` is recomputed nightly.

### Postgres — `memory_clusters` table

Batch telemetry. Columns minimally: `batch_id`, `cluster_label`,
`created_at`, and whatever else Jung writes per batch. Pruned by
`prune_memory_clusters`.

### Filesystem — `{paths.data_dir}/fade_audit/YYYY-MM.jsonl`

Monthly-rotated append-only JSONL log of state transitions, one
file per UTC year-month. One line per transition:

```json
{
  "timestamp": 1712345678.123,
  "timestamp_iso": "YYYY-MM-DDThh:mm:ss.ffffff+00:00",
  "memory_id": 4572,
  "from_state": "active",
  "to_state": "dormant",
  "reason": "nightly_recompute",
  "salience_after": 0.14
}
```

OH YEAH events include `extra` with `{source, match_strength, trigger_context_preview, days_dormant}`.

Path resolution uses the `paths.data_dir` config key (default
`/data/syn`, the Syn data root). The rotation pattern matches
`shared/nope_events.py`'s — same `_dir()` + `_current_log_path()`
shape, same `YYYY-MM.jsonl` filenames, same UTC-keyed rollover.
Bounded growth: each month's file caps at the rate of state
transitions (in steady state, on the order of tens to hundreds of
lines/day depending on memory volume).

### Source tags

```python
SOURCE_USER_CONTEXT         = "user_context"
SOURCE_DREAM_SEED           = "dream_seed"
SOURCE_DREAM_KICKER         = "dream_kicker"
SOURCE_FREE_ASSOCIATION     = "free_association"
SOURCE_CONVERSATIONAL_MATCH = "conversational_match"
```

Used in audit logs and in OH YEAH diary entries. A single shared
vocabulary so "where a retrieval came from" is consistent across
call sites.

---

## Key Flows — End to End

### Flow 1 — Memory ingestion (salience baseline set)

Not this subsystem, but worth noting for context:

```
Jung's consolidator receives a new memory
  └─ computes VAD at formation
  └─ computes vad_depth = shared.vad_math.vad_archival_depth(VADVector(...))
     (dead-zone arousal on [0.25, 0.5]; symmetric valence/dominance)
  └─ INSERT INTO memory_entries (..., vad_depth_cached, ...)
  └─ activation_state defaults to 'active' on insert
  └─ salience_cached starts NULL (set on next nightly recompute)
```

After ingest, the memory is immediately ACTIVE by default and
eligible for retrieval, even though `salience_cached` hasn't been
computed yet. The nightly recompute is the first time the memory's
effective salience is written to the row.

### Flow 2 — Nightly salience recompute

Runs once per DMN memory-review cycle (daily).

```
DMN reflection cycle reaches memory-review subphase
  └─ asyncio.to_thread(recompute_all_salience)
     └─ SELECT id, vad_depth_cached, EXTRACT(EPOCH FROM created_at),
               EXTRACT(EPOCH FROM last_recalled_ts),
               recall_count, activation_state
          FROM memory_entries
     └─ for each row:
          build MemoryRow
          salience = compute_salience(m)
          new_state = compute_activation_state(salience, prior_state)  # ← hysteresis
          if new_state != prior_state:
             collect audit event (prior_state → new_state, salience)
          update: SET salience_cached, activation_state, salience_updated_at
     └─ UPDATE via executemany in batch
     └─ COMMIT
  └─ for each collected audit event:
     write_fade_audit(memory_id, from, to, "nightly_recompute", salience_after)
```

Key ordering: fade_audit writes happen OUTSIDE the PG transaction.
A jsonl-append failure doesn't roll back the DB updates. State
transitions are the priority; audit is best-effort.

### Flow 3 — Conversational OH YEAH

Runs once per DMN reflection cycle.

```
DMN finishes generating a reflection output
  └─ calls scan_conversational_oh_yeah(reflection_output)
     └─ embed reflection text
     └─ fetch candidates from jung.query_oh_yeah_candidates
        (DORMANT + FADED only, limit 500)
     └─ cosine similarity for each
     └─ filter: similarity >= 0.95
     └─ sort descending, take top 3
     └─ for each of top 3:
        handle_oh_yeah(memory_id, reflection_text, similarity,
                       source="conversational_match",
                       content_preview=..., days_dormant=..., prior_state=...)
        → updates PG + writes diary + writes audit
     └─ returns list of event dicts
```

The `query_oh_yeah_candidates` helper in Jung filters to memories
with non-null embeddings in the target states. Limit 500 is a safety
cap — even in a very large memory store, picking up to 500
candidates per scan keeps the cost bounded and the top-3 filter
stable.

### Flow 4 — Dream kicker

Runs once per dream cycle (less frequent than memory-review).

```
DMN enters dream cycle
  └─ fetches ALL memory rows (not just dormant/faded) with embeddings
  └─ converts to MemoryRow list
  └─ sample_kicker(memory_rows)
     └─ weights = [compute_kicker_weight(m.vad_depth, m.age_days) for m in memory_rows]
     └─ weighted-random pick
  └─ chosen = kicker result
  └─ if chosen.activation_state in ("dormant", "faded"):
        handle_oh_yeah(chosen.id, dream_context, 1.0,
                       source="dream_kicker", ...)
        → full OH YEAH treatment including diary
     else:
        record_retrieval(chosen.id, source="dream_kicker")
        → just bumps recall_count, no diary (normal retrieval path)
```

The kicker is different from dream-seed retrieval in that it samples
from ALL states (active, dormant, faded). Seeds are pulled from
active memories only. The kicker is "one weighted draw that might
surface something you'd forgotten"; the seeds are "memories currently
salient enough to dream from."

`compute_kicker_weight`:

```
weight = vad_depth × recency_bias(age)
recency_bias = 0.3 + 0.7 × 2^(-age/90d)
```

- At age 0: weight = `vad_depth × 1.0`.
- At age 90: weight = `vad_depth × 0.65`.
- At age 365: weight = `vad_depth × 0.39`.
- At age → ∞: weight = `vad_depth × 0.3` (the floor).

A `vad_depth=0` memory has weight 0 regardless of age — completely
neutral memories never win the lottery. Acceptable tradeoff: if it
becomes a problem, floor `vad_depth` at 0.05 in the weight formula.

### Flow 5 — Normal retrieval

For any retrieval that lands on an ACTIVE memory through normal
query paths:

```
shared/rag.semantic_recall(query, limit=10, min_certainty=0.5)
  └─ near_vector(query_embedding, limit=limit*3) ← over-fetch
  └─ post-filter:
       drop FADED
       drop DORMANT with certainty < 0.925   (cosine 0.85)
       keep ACTIVE at min_certainty
  └─ trim to caller's limit
  └─ caller (memory_manager.semantic_find_memories) merges with keyword

[on use of an ACTIVE memory:]
  └─ record_retrieval(memory_id, source=SOURCE_USER_CONTEXT or SOURCE_DREAM_SEED)
     └─ UPDATE memory_entries
          SET recall_count = recall_count + 1,
              last_recalled_ts = NOW()
        WHERE id = memory_id
     (activation_state unchanged — stays ACTIVE until next recompute)
```

DORMANT memories surface only on a strong semantic match (cosine
≥ 0.85) — the soft-DORMANT-return path. No diary, no recall bump,
no state change; the return is a quiet "yes, you can retrieve this."
The full OH YEAH event (diary, 5× recall bump, state flip) only
fires from the conversational scan at cosine ≥ 0.95 or the dream
kicker. FADED memories are excluded from `semantic_recall`
entirely — the only path that surfaces them is the dream kicker,
which queries Postgres directly, not Weaviate.

No diary entry. No audit. Just the counter bump and timestamp. This
is the common case — most retrievals don't produce narrative events.

---

## Tunable Constants

Defined at module level in `shared/memory_salience.py` and resolved
at import time via `shared.config.get(key, default)`. The hardcoded
fallbacks below are the production defaults; override any of these
in `config/unified_config.yaml::memory_salience.*` for runtime
tuning without code edits. Existing `from shared.memory_salience
import THRESHOLD_ACTIVE` style imports continue to work — the
config-driven values are still module-level constants.

| Constant | Default | YAML key | What it controls |
|---|---|---|---|
| `BASE_HALFLIFE_DAYS` | `60.0` | `memory_salience.base_halflife_days` | Half-life for a `vad_depth=0`, `recall_count=0` memory. |
| `VAD_DEPTH_COEFFICIENT` | `2.0` | `memory_salience.vad_depth_coefficient` | Multiplier on base half-life per unit of vad_depth. |
| `RECALL_LOG_COEFFICIENT` | `1.0` | `memory_salience.recall_log_coefficient` | Multiplier on log(recall_count+1) added to halflife. |
| `RECENCY_BOOST_MAX` | `0.3` | `memory_salience.recency_boost_max` | Peak short-term bump after a recall. |
| `RECENCY_BOOST_HALFLIFE_HOURS` | `12.0` | `memory_salience.recency_boost_halflife_hours` | Half-life of the recency boost. |
| `THRESHOLD_ACTIVE` | `0.25` | `memory_salience.threshold_active` | Minimum salience for ACTIVE. |
| `THRESHOLD_DORMANT` | `0.08` | `memory_salience.threshold_dormant` | Minimum salience for DORMANT; below is FADED. |
| `HYSTERESIS_BAND` | `0.05` | `memory_salience.hysteresis_band` | Sticky band on upward transitions: a memory must clear `THRESHOLD + HYSTERESIS_BAND` to flip to a higher tier (e.g., `0.30` for DORMANT → ACTIVE). Staying in or dropping out uses the base thresholds. |
| `KICKER_RECENCY_FLOOR` | `0.3` | `memory_salience.kicker_recency_floor` | Floor on dream kicker recency bias — even very old memories can win. |
| `KICKER_RECENCY_HALFLIFE_DAYS` | `90.0` | `memory_salience.kicker_recency_halflife_days` | Recency-bias half-life for kicker weighting. |
| `OH_YEAH_RECALL_BUMP` | `5` | `memory_salience.oh_yeah_recall_bump` | Recall-count increment on OH YEAH. |
| `OH_YEAH_CONVERSATIONAL_THRESHOLD` | `0.95` | `memory_salience.oh_yeah_conversational_threshold` | Cosine threshold for conversational OH YEAH. |
| `OH_YEAH_DORMANT_SOFT_THRESHOLD` | `0.85` | `memory_salience.oh_yeah_dormant_soft_threshold` | Cosine threshold for soft DORMANT return (retrieval layer). |
| `OH_YEAH_MAX_RETURNS_PER_SCAN` | `3` | `memory_salience.oh_yeah_max_returns_per_scan` | Cap on OH YEAH events per conversational scan. Lives in `shared/memory_recall.py` (it's a scan-loop parameter, consumer-side, rather than a pure-math salience tunable) but shares the YAML namespace with the other OH YEAH tunables for operator convenience. |

Relationships between constants (useful for proposing changes):

- Raising `BASE_HALFLIFE_DAYS` makes everything stick longer
  proportionally; it rescales the whole distribution.
- Raising `VAD_DEPTH_COEFFICIENT` widens the gap between mundane and
  emotionally-weighty memories — the latter stick much longer
  relatively.
- Raising `THRESHOLD_DORMANT` above 0.1 would push low-depth fresh
  memories directly to FADED on ingestion. Not what you want.
- Lowering `OH_YEAH_CONVERSATIONAL_THRESHOLD` would fire more OH YEAH
  events per scan — more narrative color but risk of noise.

---

## Invariants and Contracts

1. **`memory_salience` has no I/O.** Pure math, safe to import
   anywhere, safe to unit-test. Changes to storage schema don't
   affect this module.
2. **All state transitions go through `recompute_all_salience` OR
   `handle_oh_yeah`.** No other writers of `activation_state`
   exist. `record_retrieval` deliberately does not change state —
   only `recall_count` and `last_recalled_ts`.
3. **OH YEAH is the only direct-promotion path to ACTIVE.** Normal
   retrieval doesn't transition DORMANT → ACTIVE. The memory stays
   DORMANT unless nightly recompute promotes it based on updated
   salience.
4. **Physical deletion is not performed.** Faded memories stay in the
   database with `activation_state='faded'`; their query surface
   shrinks, but the rows persist. Any deletion is a separate
   admin-opt-in concern.
5. **`fade_audit/YYYY-MM.jsonl` files are append-only.** The
   monthly-rotated audit trail is never rewritten in place. Old
   month files persist; rotation happens by file selection at
   write time, not by moving content.
6. **Recency boost only applies if there's a `last_recalled_ts`.**
   Never-recalled memories get `days_since_recall=None` and the boost
   is zero. They get no "fresh recall" pop.
7. **Kicker is weighted but not deterministic.** A `vad_depth=0`
   memory has weight 0, no lottery. Otherwise sampling is probabilistic
   — any memory with non-zero weight can be picked.
8. **OH YEAH scan is capped at 3 returns.** If more memories match the
   threshold, they stay DORMANT/FADED this cycle. Next cycle's scan
   against a different context may pick different matches.
9. **Failure modes are non-fatal.** Every function catches broad
   exceptions at appropriate boundaries and returns a sensible
   default (empty list, False, None). The salience system degrades
   gracefully — if PG is down, retrieval keeps working, just without
   state tracking.

---

## Integration Points

### Inbound (who calls this subsystem)

| Caller | What it uses |
|---|---|
| `services/dmn/scheduler.py` MEMORY_REVIEW phase handler | `recompute_all_salience`, `prune_memory_clusters` |
| `services/dmn/scheduler.py::_gather_dream_seeds` (dream kicker block) | `MemoryRow`, `sample_kicker`, `age_days`, `days_since`, `handle_oh_yeah`, `record_retrieval`, source tags |
| `services/dmn/scheduler.py` REFLECTION phase handler (conversational scan after reflection output) | `scan_conversational_oh_yeah` |
| `services/jung/consolidator.py::pg_query_by_user` (per-user retrieval bump) | `record_retrieval(SOURCE_USER_CONTEXT)` |
| `services/jung/consolidator.py::pg_query_high_arousal` (dream seed bump) | `record_retrieval(SOURCE_DREAM_SEED)` |
| `tests/test_memory_retention_flow.py` | Full surface for integration tests |

### Outbound (what this subsystem calls)

- `services.jung.consolidator._get_pg` — Postgres pool accessor.
- `services.jung.consolidator.query_oh_yeah_candidates` — fetches
  candidates for the conversational OH YEAH scan.
- `shared.utils.write_diary_entry` — OH YEAH diary entries.
- `shared.rag._embed` — embedding for conversational scan.
- `shared.rag.update_memory_state` — mirrors Postgres
  `activation_state` transitions into Weaviate so retrieval can
  filter by tier. Fired from both `recompute_all_salience` (per
  state-transition event) and `handle_oh_yeah` (DORMANT/FADED →
  ACTIVE). Best-effort; failures are logged and the next recompute
  resyncs.
- `shared.config.get` — `paths.data_dir` for fade_audit location.

Interesting: `memory_recall` imports from `services.jung.consolidator`
at call time rather than at module import, which preserves the
convention that `shared/` doesn't hard-depend on `services/`. The
`_get_pg` call is lazy and fails cleanly if Jung isn't up. The same
late-import pattern applies to `shared.rag.update_memory_state` for
the Weaviate mirror — `memory_recall` doesn't hard-depend on RAG
either, so a Weaviate-less smoke test still exercises the
salience/state pipeline cleanly.

### The Postgres → Weaviate state mirror

Postgres `memory_entries.activation_state` is the source of truth.
Weaviate carries a derived copy of the same value on every
`SynMemory` object (under the `activation_state` property) so that
retrieval — which goes through Weaviate's `near_vector` —  can
filter by tier without round-tripping to Postgres.

Two writers keep Weaviate in sync:

- `recompute_all_salience` — after the PG transaction commits,
  iterates the audit-event list (one event per state transition)
  and calls `update_memory_state(filename, new_state)` for each.
  Memories whose state didn't change don't generate any Weaviate
  traffic. Sync volume is bounded by transition rate, not memory
  count.
- `handle_oh_yeah` — mirrors the immediate DORMANT/FADED → ACTIVE
  flip so a freshly-promoted memory is reachable via the ACTIVE
  retrieval path on the next call, rather than waiting for the next
  nightly recompute. Filename comes back from the existing PG
  UPDATE via `RETURNING filename`.

Both writers fail-soft: Weaviate-down or object-missing logs at
`debug` and continues. A dropped Weaviate update means retrieval
may briefly see stale state for that memory until the next
recompute pass corrects it.

---

## Related Docs

- `MEMORY_ARCHITECTURE.md` — conceptual parent. Covers memory types,
  file format, consolidation cycle, dream distillation, the overall
  shape of memory. This doc focuses specifically on the salience
  model and runtime state management.
- `DESIGN_DECISIONS.md` §P *VAD Consolidation* — `VADVector` is what
  Jung uses to compute `vad_depth_cached` at ingestion.
- `services/jung/consolidator.py` module header — the owner of the
  Postgres schema this subsystem reads from. Jung is load-bearing
  for the ingestion path; this subsystem is the state-tracking layer
  on top.
- `services/dmn/scheduler.py::_gather_dream_seeds` — dream cycle
  consumer of this subsystem's kicker sampler.
- `services/dmn/scheduler.py` REFLECTION phase handler — consumer of
  `scan_conversational_oh_yeah`.
- `CHANGELOG.md` — memory salience and recall mechanics were
  introduced in a specific phase; the narrative there has useful
  motivational context this reference doc doesn't carry.
