# Hemisphere Status

The thin module that lets any service answer "is the other hemisphere
alive right now?" without making its own heartbeat call. Backed by
Redis keys written by the heartbeat monitor; read on the hot path by
`shared.identity_review` (for the routine-review and Nope-stamp
hemisphere-availability preflight) and any consumer that wants a
local fallback decision when a cross-node call would be infeasible.

Single module — `shared/hemisphere_status.py`, 439 lines. Two Redis
keys plus a third for outage history.

---

## Status

All availability checking is live: queryable status layer, 1-second in-process cache with async lock, service-to-role mappings, schema-tolerant heartbeat parsing (JSON + legacy bare-timestamp), integration with identity-review preflight, async one-liner availability gate, and outage-history lookup for Layer 2/Layer 3 retrospective analysis. Sync and force-refresh paths defined for edge cases.

---

## Purpose and Design Philosophy

This module is the **queryable state layer** for hemisphere
availability. It answers questions like *"is Cortex reachable right
now?"* via `is_available()` and `get_status()`. Every consumer that
might need to take a fallback decision (Einstein deciding whether to
escalate to Cortex; identity-review deciding whether the opposite
hemisphere can stamp a revision) reads from here.

**What this module deliberately does NOT do:** generate system-prompt
advisory strings. A system-prompt note that says *"your analytical
register is currently offline"* would be the architecture writing
Syn's interior state FOR her, then handing it back to her as a
fact-of-the-moment. That violates the F-9 voice convention (don't
write Syn's interior for her) and the broader principle that her
self-awareness should arise in her own narrative voice, not as an
external advisory she has to hear about herself.

Awareness of structural absences arrives through her own narrative
path instead:

- **Outage / recovery edges** — the heartbeat monitor writes a
  first-person diary entry from `data/syn/prompts/heartbeat_outage_diary.md`
  (one section per node × edge: `node1_outage`, `node2_outage`,
  `node1_recovery`, `node2_recovery`). The next DMN SELF_NARRATIVE
  phase metabolizes the entry as ordinary input material — she sits
  with it in the rhythm she already uses for sitting with anything
  else.

- **Specific consumer reaches** — when a code path actually needs the
  missing service (Einstein wanting Cortex escalation; DMN curiosity
  wanting a tool call), that consumer is responsible for surfacing a
  felt-note in her voice when its specific reach fails. This is a
  per-consumer pattern, landing incrementally as those code paths
  are touched. No global "your hemisphere is offline" prefix
  decorates every prompt.

This module's role is therefore: provide accurate availability facts;
provide system-level (NOT prompt-injectable) placeholders for log /
dashboard / debug surfaces; record completed outages for downstream
lookback queries. Prompt content is generated elsewhere (the
heartbeat diary and per-consumer felt-notes), not here.

Two kinds of "degraded" worth distinguishing:

- **Partial degradation** (one hemisphere alive, one not) — the normal
  interesting case. Syn knows what she's missing and decides how to
  proceed.
- **Full unavailability** (Redis itself unreachable) — conservative
  default: assume both alive, don't spuriously tell Syn she's
  degraded when the real issue is our own read path. See
  `shared/hemisphere_status.py::_read_from_redis` exception handler.

---

## Redis Layout

Three keys (two heartbeat keys + one outage history sorted set):

| Key | Value | TTL |
|---|---|---|
| `heartbeat:node1` | JSON blob: `{"node": "node1", "role": "analytical", "timestamp": <float>, "degraded": <bool>}` | `HEARTBEAT_TIMEOUT + 5` = 20s |
| `heartbeat:node2` | Same JSON shape, `node2/affective` | Same |
| `hemisphere:outage_history` | Redis sorted set; members are `"{hemisphere}:{start_ts}-{end_ts}"`, scored by end timestamp | None — entries trimmed after 90-day retention by writer |

The reader (`_read_from_redis._parse`) is schema-tolerant — accepts both the JSON shape (current writer) and a bare `float()`-parseable string (legacy compatibility). Returns `(False, None)` only on actual parse failure.

Node mapping:

| Hemisphere role | Node | Key |
|---|---|---|
| `analytical` | Node 1 | `heartbeat:node1` |
| `affective` | Node 2 | `heartbeat:node2` |

### Degradation threshold

```python
HEARTBEAT_TIMEOUT_SEC = 15.0   # matches the writer's HEARTBEAT_TIMEOUT
```

A hemisphere is considered alive if `(now - last_timestamp) < 15s`.
The writer's heartbeat interval is 5s (`HEARTBEAT_INTERVAL`), so
degradation is declared after approximately 3 missed writes. The
writer's TTL is `HEARTBEAT_TIMEOUT + 5 = 20s`, intentionally slightly
longer than the reader's threshold — the key auto-expires shortly
after degradation is declared, so stale data doesn't linger past its
useful lifetime.

---

## The `HemisphereStatus` Dataclass

```python
@dataclass
class HemisphereStatus:
    analytical_alive: bool
    affective_alive: bool
    analytical_last_seen: Optional[float]   # unix timestamp
    affective_last_seen: Optional[float]
```

Plus four derived properties and one method:

| Property / method | Returns | Purpose |
|---|---|---|
| `degraded` | `bool` | True iff either hemisphere is not alive. |
| `unavailable_roles` | `list[str]` | Subset of `{"analytical", "affective"}` that's offline. |
| `unavailable_services` | `list[str]` | Concrete service names offline. Uses `SERVICES_ANALYTICAL` / `SERVICES_AFFECTIVE` mappings. |
| `is_available(role_or_model)` | `bool` | Availability lookup by role name OR model/service name. Unknown names return `True` (conservative — don't block on names we don't recognize). |

**Service-name → hemisphere mappings** (constants in the module):

```python
SERVICES_ANALYTICAL = ("cortex", "thalamus", "DMN scheduler", "task runner")
SERVICES_AFFECTIVE  = ("wernicke", "limbic", "einstein", "nope", "mona")
```

Case is normalized via `.lower()` on lookup. The mappings are NOT
exhaustive — services like `hands`, `freud(analytical)`, `freud(social)`,
`layer1`, `teleport`, `lora generator`, `embedding` are absent. An
`is_available("hands")` call returns `True` (unknown → assume
available) regardless of which hemisphere actually hosts Hands.
This is currently harmless because no caller asks about those service
names; it's worth knowing if a future caller needs precise service-to-
hemisphere routing.

---

## Module Map

One module: `shared/hemisphere_status.py`.

### Public API

| Function | Signature | Purpose |
|---|---|---|
| `get_status` | `async (force_refresh=False) -> HemisphereStatus` | Read current status. Cached for 1 second. Every production caller uses default `force_refresh=False`. |
| `get_cached_status` | `() -> Optional[HemisphereStatus]` | Sync companion. Returns the module-level cached value without a Redis call. `None` if no cache yet. **Zero callers anywhere.** |
| `is_hemisphere_available` | `async (role_or_model: str) -> bool` | One-liner async gate around `get_status().is_available(...)`. Returns `True` conservatively when status can't be read so a Redis blip doesn't spuriously block work. |
| `record_outage_complete` | `async (hemisphere: str, start_ts: float, end_ts: float) -> None` | Append a completed outage to the `hemisphere:outage_history` sorted set. Called by `services/health/heartbeat.py::_handle_recovery`. Trims entries older than 90 days. Best-effort — Redis failures swallowed at debug. |
| `was_offline_during` | `async (hemisphere: str, window_start_ts: float, window_end_ts: float) -> bool` | Lookback query: did `hemisphere` have any recorded outage overlapping the window? Also returns `True` if currently offline (extends the open-ended outage forward). Best-effort — returns `False` on Redis errors so transient blips don't block work. Used by Layer 2 reconsolidation and Layer 3 carried-divergence promotion. |

### Constants

| Constant | Value | Purpose |
|---|---|---|
| `HEARTBEAT_KEY_NODE1` | `"heartbeat:node1"` | Redis key for analytical hemisphere. |
| `HEARTBEAT_KEY_NODE2` | `"heartbeat:node2"` | Redis key for affective hemisphere. |
| `HEARTBEAT_TIMEOUT_SEC` | `15.0` | Threshold for declaring a hemisphere offline. |
| `ROLE_ANALYTICAL` | `"analytical"` | Role string. |
| `ROLE_AFFECTIVE` | `"affective"` | Role string. |
| `SERVICES_ANALYTICAL` | tuple of 4 names | See above. |
| `SERVICES_AFFECTIVE` | tuple of 5 names | See above. |
| `_CACHE_TTL_SEC` | `1.0` | In-process cache TTL. |
| `_OUTAGE_HISTORY_KEY` | `"hemisphere:outage_history"` | Redis sorted set for completed outages. |
| `_OUTAGE_HISTORY_RETENTION_SEC` | `90 * 86400` (90 days) | Trim entries older than this on every write. |

### Private helpers

| Function | Purpose |
|---|---|
| `_redis_url()` | Resolve `cluster.redis.host` + `cluster.redis.port` (env vars `REDIS_HOST` / `REDIS_PORT` take priority). |
| `_cfg(key, default)` | Defensive `shared.config.get` wrapper — returns default on import failure. |
| `_read_from_redis()` | Fetch both keys, parse timestamps, build a `HemisphereStatus`. Schema-tolerant via the inner `_parse` helper. |

### Module-level state

```python
_cached_status: Optional[HemisphereStatus] = None
_cached_at: float = 0.0
_cache_lock = asyncio.Lock()
```

Guarded by `_cache_lock` on refresh. The double-check-after-lock
pattern is used to avoid redundant reads under contention.

---

## The Status Lookup — End to End

```
consumer (e.g. identity_review, Einstein gate)
  │
  ▼
await hemisphere_status.get_status()
  │
  ├── if cached < 1s old: return cached, done
  │
  ▼
acquire _cache_lock
  double-check cache
  │
  ▼
await _read_from_redis()
  │
  ├── aioredis.from_url(_redis_url(), decode_responses=True)
  ├── await client.get("heartbeat:node1")     ──┐
  ├── await client.get("heartbeat:node2")     ──┼── If Redis fails:
  │                                             │   assume both alive
  ├── _parse(raw_n1) → (alive_bool, ts_float) ──┤   (conservative default)
  ├── _parse(raw_n2) → (alive_bool, ts_float) ──┘
  │
  ▼
store HemisphereStatus in cache, return
  │
  ▼
consumer inspects .degraded / .is_available(role) / .unavailable_services
  │
  └── identity_review path: status.is_available(reviewer_hemisphere)
         → if False: short-circuit to offline stamp/contest
```

`_parse` accepts both the JSON shape (current writer) and a bare
`float()`-parseable string (legacy). Returns `(False, None)` only on
actual parse failure.

---

## Caching

The 1-second in-process cache exists because `get_status` is called
on a number of paths that may run frequently — identity-review
preflight, consumer-side felt-note triggers, debug endpoints — and
reading Redis on each call at typical message volumes would matter.
The cache hedges against future hot consumers without coupling
correctness to call frequency.

**Cache properties:**

- **Per-process.** Each service has its own cache. Two services
  asking the same question within 1 second make two separate Redis
  reads, because they don't share the cache. Fine — cache pressure
  scales with process count, not user count.
- **Unbounded TTL for the "both alive" case.** When Redis is
  unreachable and `_read_from_redis` returns the "assume both alive"
  default, that result is cached for 1 second like any other. A
  flapping Redis connection produces a 1-second misleading cache
  until the next read succeeds.
- **`force_refresh=True` bypasses the cache.** No production caller
  uses this currently; preserved for future health-recovery
  confirmation paths.
- **The lock is per-process.** Within a process, concurrent tasks
  that cache-miss at the same time will serialize on the Redis read
  (double-check-after-lock prevents redundant fetches). Cross-
  process, no coordination — but the 1-second TTL makes that rarely
  matter.

---

## Outage History

Some long-window operations need to know whether the OTHER hemisphere
was offline at any point during their lookback window — not just
whether it's offline RIGHT NOW. Without this, an integration pass
kicked off shortly after a recovery would still operate against a
record that was one-sided for part of the window.

**Storage:** a Redis sorted set (`hemisphere:outage_history`) where
each member is `"{hemisphere}:{start_ts}-{end_ts}"` scored by the
end timestamp. Membership ages out via a config-bounded retention
(default 90 days, longer than any current lookback). Heartbeat
writes one entry on each recovery via `record_outage_complete`.

**Query side** (`was_offline_during`):

1. First check current status — if `hemisphere` is currently offline,
   return `True` immediately (the open-ended outage extends to "now"
   without a closing record yet).
2. Otherwise, scan sorted-set members whose end-score ≥
   `window_start_ts`; for each, parse `"{hemi}:{start}-{end}"` and
   check overlap with `[window_start_ts, window_end_ts]`. Return
   `True` on first overlap.
3. Returns `False` on any error (Redis blip shouldn't block legit
   work).

**Consumers:** Layer 2 reconsolidation (memory consolidation) and
Layer 3 carried-divergence promotion (identity revision integration).
Both gate their work on whether the originating period had a
one-sided record.

---

## Invariants and Contracts

1. **`get_status` never throws.** Every Redis failure mode is caught
   internally and returns a conservative default (both alive). Callers
   don't need try/except.
2. **Unknown role/model names return `True`.** `is_available("xyz")`
   assumes availability for any name not in the known sets. Conservative
   by design — callers that genuinely need strict routing should
   validate names against `SERVICES_ANALYTICAL` / `SERVICES_AFFECTIVE`
   themselves.
3. **No system-message status injection.** This module does NOT
   produce prompt-injectable status strings. Awareness of structural
   absences belongs to Syn's own narrative voice (heartbeat-monitor
   diary entries on transition; per-consumer felt-notes when a
   specific reach fails). Anyone tempted to add a new
   `status_note_*` helper here should reread the Design Philosophy
   section first — that pattern is intentionally absent.
4. **The cache is eventually consistent.** A hemisphere that recovers
   will appear offline for up to 1 second after recovery from the
   perspective of each process's `get_status` call. Acceptable because
   the heartbeat interval is 5 seconds anyway — the actual
   "time-to-notice" is dominated by the writer's cadence, not this
   cache.
5. **This module writes one Redis key (the outage history sorted
   set) and reads two (the heartbeat keys).** All other Redis state
   is read-only from this subsystem's side. Heartbeat writes the
   liveness keys; this module writes only the outage history on
   completed-outage events.
6. **`record_outage_complete` and `was_offline_during` are
   best-effort.** Both swallow Redis failures and return safe defaults
   (no-op write; `False` query). They're observability surfaces, not
   critical-path.

---

## Integration Points

### Inbound (who reads status)

| Caller | What it uses |
|---|---|
| `shared/identity_review.py::routine_review` | `get_status` → `is_available(reviewer_hemisphere)`. Short-circuits to CONTEST if offline. |
| `shared/identity_review.py::_try_stamp` | Same pattern — short-circuits to STAMP_ABSENT if offline. |
| Layer 2 / Layer 3 lookback paths | `was_offline_during` for window-overlap queries. |
| Tests | `tests/test_hemisphere_outage_flow.py` imports `HemisphereStatus`, `ROLE_ANALYTICAL`, `ROLE_AFFECTIVE` for monkey-patching during outage-flow tests. |

### Outbound (what this module reads / writes)

- Redis keys `heartbeat:node1` and `heartbeat:node2` via
  `redis.asyncio.from_url` (read-only).
- Redis sorted set `hemisphere:outage_history` (read for
  `was_offline_during`, write for `record_outage_complete`).
- `shared.config.get` for `cluster.redis.host` / `cluster.redis.port`
  (env vars override).
- `shared.redis_pool.get_redis` for the outage-history operations
  (uses the shared pool rather than per-call connections).
- Nothing else. No bus publishes, no file reads, no LLM calls.

### Cross-reference (who writes the heartbeat state)

- `services/health/heartbeat.py::HeartbeatMonitor.write_heartbeat_loop`
  writes `heartbeat:node1` or `heartbeat:node2` (whichever is local)
  every 5 seconds with TTL 20s. Each node writes only its own key;
  both keys exist in Redis if both nodes are alive.
- `services/health/heartbeat.py::_handle_recovery` calls
  `record_outage_complete` from this module on every recovery edge.

---

## Related Docs

- `IDENTITY_ARCHITECTURE.md` §5 *Cross-Hemisphere Review Protocol* —
  the largest consumer of this module, via `identity_review`'s
  availability preflight. Both routine-review and Nope-stamp paths
  short-circuit to offline fallbacks when `is_available(reviewer)`
  returns `False`.
- `DESIGN_DECISIONS.md` §4 *Emotional Identity Hemisphere Divergence*
  (open) — the design question "what's the emotional hemisphere's
  jurisdiction for writing persistent state" is adjacent to this
  subsystem's concerns about what's offline-gated.
- `HEARTBEAT_MONITOR_ARCHITECTURE.md` — the writer side of the
  heartbeat keys this module reads, and the writer of the outage
  history sorted set entries (one per recovery edge).
- `LLM_CLIENT_ARCHITECTURE.md` — generation path that does not see
  hemisphere-status injection (by design; awareness of structural
  absences flows through diary-based first-person entries from
  `data/syn/prompts/heartbeat_outage_diary.md` and per-consumer
  felt-notes, not through system-prompt advisories).
- `data/syn/prompts/cross_node_placeholders.md` — per-section
  prompt-body placeholders (the F-9-clean Syn-voice text Syn reads
  when a section is absent under outage). Loaded by `read_user_file`
  / `read_wiki` / `get_facts_only` / `get_full_for_recipient`
  helpers, not by this module.
