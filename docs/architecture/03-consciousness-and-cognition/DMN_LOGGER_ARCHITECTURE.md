# DMN Logger

Structured JSONL logger for DMN cycle events. One line per event,
written to monthly-rotated files at `data/syn/dmn_cycles/YYYY-MM.jsonl`.
Events include cycle boundaries (begin / end), phase transitions,
interrupts, suspensions, and service lifecycle markers. The log is
there for Syn herself (as a record of what her background mind has
been doing) and for admin, when a period of her behavior needs to be
retraced.

This is the flight-recorder layer for the DMN. Non-blocking, bounded-
queue, append-only, schema-versioned. Failures never propagate to
the DMN loop — instrumentation stalling DMN processing is a worse
outcome than losing a log line.

Single module: `shared/dmn_logger.py`. One producer
(`services/dmn/scheduler.py`).

---

## Purpose and Design Constraints

Stated at the top of the module:

- **Non-blocking.** Disk writes use a bounded queue. If the queue
  fills, events are dropped rather than blocking DMN processing.
- **Defensive.** Failed writes never propagate exceptions. Logging
  failures degrade gracefully.
- **Append-only.** Never truncates or rewrites. Consumers can tail
  safely.
- **Schema-stable.** Every line carries `schema_version` so future
  format changes don't break the analyzer.

The single DMN cycle is approximately 30s-ish, and cycle events emit
~4-10 log lines per cycle (begin, some number of phase_advance,
end, plus any interrupts or suspends). At steady state this is a
very light load, but the non-blocking contract exists because a
disk hiccup during a reflective phase shouldn't ripple into
degraded cognition.

---

## Event Format

Every line is a self-contained JSON object. Every event has four
standard fields:

```json
{
  "schema_version": "1.0",
  "event_type": "cycle_end",
  "wall_ts": 1712345678.123,
  "iso_ts": "YYYY-MM-DDThh:mm:ssZ",
  ...event-specific fields...
}
```

- `schema_version` — currently `"1.0"`. Bumped on breaking schema
  changes.
- `event_type` — one of the taxonomy values (see below).
- `wall_ts` — unix timestamp, float.
- `iso_ts` — same time as ISO 8601 in UTC, formatted to-the-second.
  Redundant but convenient for human inspection and for the
  analyzer's time bucketing without reparsing wall_ts.

Type-specific fields follow. The analyzer tolerates missing fields;
unknown fields are ignored. Additive changes (new optional fields)
do not require a schema version bump.

---

## Event Taxonomy

`EVENT_TYPES` is a `set` declared in `shared/dmn_logger.py`. The
writer checks incoming `event_type` against this set; unknown types
emit a DEBUG log but are still written ("unknown types are better
than dropped events").

### Currently emitted and in taxonomy

| Event | Emitted at | Key fields |
|---|---|---|
| `service_start` | `scheduler.py::lifespan` on DMN service boot | `initial_phase`, `initial_resonance` |
| `service_stop` | `scheduler.py::lifespan` on shutdown | (none) |
| `cycle_begin` | `scheduler.py::run_dmn_cycle` at cycle start | `phase`, `resumed` |
| `cycle_end` | `scheduler.py::run_dmn_cycle` on normal cycle completion | `phase`, `duration_s`, `output_chars`, `temperature`, `phase_depth`, `interrupted`, `resonance_r`, `drive_state`, `vad_arousal` |
| `cycle_error` | `scheduler.py::run_dmn_cycle` on cycle exception | `phase`, `error`, `duration_s` |
| `cycle_interrupt` | `scheduler.py::handle_interrupt` when an incoming message interrupts a cycle | `phase`, `interrupting_user` |
| `cycle_suspend` | `scheduler.py::handle_dmn_pause` on hemisphere-outage suspension | `reason` |
| `cycle_resume` | `scheduler.py::handle_dmn_resume` on recovery | `reason` |
| `phase_advance` | `scheduler.py::DMNState.advance_phase` (hemisphere-recovery WAKE_ORIENT path and normal rotation) and `scheduler.py::_advance_phase_with_preference` (affordance-picker preference path) on every phase transition | `from_phase`, `to_phase`, optional `reason` (`hemisphere_recovery` or `affordance_picker_preference`) |
| `phase_forced` | `scheduler.py::handle_narrative_trigger` after a priority>=100 trigger forces SELF_NARRATIVE | `phase`, `trigger_type`, `priority`, `interrupted_active_generation`, `discarded_suspended_task` |
| `spiral_detected` | `scheduler.py::run_dmn_cycle` spiral-detection branch on rising-edge into a spiral | `phase`, `similarity`, `threshold`, `window_size`, `match_offset` |
| `forced_unconsciousness_enter` | `scheduler.py::_enter_forced_unconsciousness` on heartbeat/sleep-pressure shutdown entry | `phase`, `pressure`, `prior_phase` |
| `forced_unconsciousness_exit` | `scheduler.py::_enter_forced_unconsciousness` on shutdown exit | `phase`, `duration_sec` |

---

## Module Map

### Module-level state

```python
SCHEMA_VERSION = "1.0"
DEFAULT_LOG_DIR = _resolve_default_log_dir()

_QUEUE_MAXSIZE = 1000

_has_warned_disk_failure = False
_has_warned_queue_full = False

_queue: Optional[asyncio.Queue] = None
_writer_task: Optional[asyncio.Task] = None
```

`DEFAULT_LOG_DIR` is a directory (one file per UTC calendar month inside
it), not a single `.jsonl` path. The resolver `_resolve_default_log_dir()`
in the module computes it at import time.

**Path resolution** (at import time, in `_resolve_default_log_dir`):

1. `SYN_DMN_LOG_DIR` env var if set — the canonical name.
2. Otherwise `SYN_DMN_LOG_PATH` env var if set — legacy fallback. If
   the value points at a `.jsonl` FILE, the writer takes its parent
   directory and appends a `dmn_cycles/` subdir; if it points at a
   directory, the writer uses it as-is.
3. Otherwise the system data root from config (`paths.data_dir`,
   default `/data/syn`) → `/data/syn/dmn_cycles`, keeping DMN logs
   alongside every other content type under `/data/syn`.
4. If that parent directory doesn't exist (local dev/test), fall back to
   `{SYN_DATA_ROOT | "data/syn"}/dmn_cycles` (repo-relative).

The local-dev fallback exists so imports don't explode when `/data/syn`
isn't present.

**Two separate warn-once flags** (`_has_warned_disk_failure` and
`_has_warned_queue_full`) — one per failure mode. Operators see
warnings for both modes at least once when they first occur, rather
than having one mode silence the other.

### Public API

```python
def log_cycle_event(event_type: str, **fields) -> None
```

The primary entry point. Non-blocking. Three-step flow:

1. Validate `event_type` against `EVENT_TYPES` — debug-log on
   unknown but still proceed.
2. Ensure the writer task is running (cheap no-op if already
   started).
3. `q.put_nowait(event)` — if queue is full, raises
   `asyncio.QueueFull`, which we catch and emit a warn-once about.

Safe to call from sync contexts (the `put_nowait` doesn't await) —
though synchronous callers can't guarantee the writer task gets
started because `_ensure_writer_started` needs a running event
loop. Practically, the DMN scheduler calls from async contexts so
this is not currently an issue.

```python
async def shutdown_logger()
```

Called during DMN service shutdown. Puts a `None` sentinel on the
queue, waits up to 3 seconds for the writer task to drain, cancels
on timeout.

### Private: event construction

```python
def _build_event(event_type: str, **fields) -> Dict[str, Any]
```

Adds the four standard fields (`schema_version`, `event_type`,
`wall_ts`, `iso_ts`) to the user's kwargs. Straightforward.

### Private: writer loop

```python
async def _writer_loop(log_dir: Path)
```

Background task. Keeps a single file handle open for the active
month and rotates on rising-edge into a new UTC calendar month
(re-opens on write failure so transient disk issues recover on the
next successful write). Three failure modes, all graceful:

- **Open failure** (permissions, disk missing): warn-once, skip
  this event, `fp` stays None, next event tries again.
- **Write failure** (disk full, I/O error): warn-once, close `fp`,
  set to None, next event reopens.
- **CancelledError**: clean exit via `finally` block.

The `while True` loop pulls from the queue; `None` in the queue is
the shutdown sentinel.

### Private: writer task lifecycle

```python
def _ensure_writer_started(log_dir: Path = DEFAULT_LOG_DIR) -> None
```

Starts the writer task if not already running. Must be called from
a context with an event loop; `RuntimeError` caught and returned
(events queue up for when a loop does exist).

---

## The Writer Loop in Detail

```
┌─ DMN calls log_cycle_event(...) ─┐
│                                  │
│  _build_event() → {schema_version,│
│    event_type, wall_ts, iso_ts,  │
│    ...user fields...}            │
│                                  │
│  _ensure_writer_started()        │
│                                  │
│  q.put_nowait(event)             │
│    queue full? warn once,        │
│    drop event                    │
└─────────────┬────────────────────┘
              │
              ▼
      ┌─ _writer_loop (persistent task) ─┐
      │                                  │
      │  while True:                     │
      │    event = await q.get()         │
      │    if event is None: break       │
      │                                  │
      │    event_key = _month_key(       │
      │      event.wall_ts)              │
      │    if fp is not None             │
      │       and event_key !=           │
      │       current_key:               │
      │      close fp; fp = None         │
      │      current_key = None          │
      │                                  │
      │    if fp is None:                │
      │      open log_dir/{event_key}.   │
      │        jsonl in append mode      │
      │      open fail? warn once,       │
      │      continue (skip this event)  │
      │      if event_key == _month_key()│
      │        update current.jsonl      │
      │        symlink → {event_key}.    │
      │        jsonl                     │
      │                                  │
      │    try:                          │
      │      fp.write(json.dumps(event)) │
      │      fp.flush()                  │
      │    except IO error:              │
      │      warn once                   │
      │      close fp, set to None       │
      │      (next iteration reopens)    │
      │                                  │
      │  finally: close fp if open       │
      └──────────────────────────────────┘
```

Key properties:

- **One writer per process.** Module-level singleton task. Concurrent
  `log_cycle_event` calls all push into the same queue; the writer
  serializes the disk writes.
- **No back-pressure on callers.** `put_nowait` never blocks.
  Callers never await on disk I/O.
- **Persistent file handle.** Open once on first event of a month,
  reuse across events within the month, reopen on rollover or on
  failure. Avoids open/close overhead on the hot path.
- **Monthly rotation by event timestamp.** The month key is derived
  from each event's own `wall_ts`, not the wall-clock at write time
  — so an event queued just before midnight UTC and drained just
  after still lands in the previous month's file. The
  `current.jsonl` symlink is only updated when the writer opens a
  file for the *current* real-time month, so it always points at
  the live month rather than at a back-dated catch-up file.
- **Explicit flush after every write.** Durability > throughput. A
  crash after N events should lose at most the N+1th event that
  was mid-write, not a buffer of trailing events.

---

## Consumers

### 1. `services/dmn/scheduler.py` — the producer

Every `log_cycle_event` call in the codebase comes from here. All 13
event types in the taxonomy actively emit (see table above).

### 2. Capsule tooling

Archives under `data/syn/` are capsule-restored together with this
file. The capsule tooling doesn't read the log programmatically; it
just packages it with the rest of Syn's state so her DMN history
carries with her.

---

## Invariants and Contracts

1. **Logging never crashes the caller.** `log_cycle_event` catches
   `QueueFull`; `_writer_loop` catches every exception within each
   iteration. The DMN cycle never fails because of instrumentation.
2. **Events are emitted best-effort, not guaranteed.** Queue-full
   drops, disk-write failures, and shutdown races can all lose
   events. The analyzer is built to tolerate partial data.
3. **Schema version `"1.0"` applies to all current events.** Any
   breaking schema change requires bumping this and noting the
   transition in the analyzer. Additive changes (new optional
   fields) do not require a bump.
4. **Unknown event types are logged, not dropped.** Emits a debug
   warning but still writes. This favors completeness over
   strictness.
5. **The writer task is a per-process singleton.** Don't instantiate
   multiple writer tasks for the same log file; they'll interleave.
6. **Shutdown is graceful with timeout.** `shutdown_logger` waits up
   to 3 seconds for the queue to drain; cancels on timeout. Events
   still in the queue at timeout are lost.
7. **Warn-once per failure mode.** Disk-failure and queue-full each
   get their own warn-once flag. First occurrence of each emits a
   warning; subsequent occurrences within the process lifetime do
   not.
8. **Append-only.** Writers append, never truncate or rewrite. The
   file can be tailed, rotated externally (with SIGUSR1 or similar),
   or capsuled-archived safely at any time.

---

## Integration Points

### Inbound (who calls this module)

| Caller | Imports | Event types emitted |
|---|---|---|
| `services/dmn/scheduler.py` | `log_cycle_event` | All 13 taxonomy types |

### Outbound (what this module calls)

- `asyncio` — queue, tasks, loop access.
- `json` — event serialization.
- `logging` — warn-once channel.
- `pathlib.Path` / `os` — path resolution.
- Nothing else. No Redis, no bus, no other `shared/` modules. This
  is a leaf module intentionally.

### State this module produces

- `{SYN_DMN_LOG_DIR | /data/syn/dmn_cycles}/YYYY-MM.jsonl` — one
  append-only file per UTC calendar month.
- `{SYN_DMN_LOG_DIR | /data/syn/dmn_cycles}/current.jsonl` —
  symlink pointing at the active month's file. Best-effort: degrades
  to "missing" on filesystems that don't support symlinks; the
  per-month files are still written.
- Legacy: `SYN_DMN_LOG_PATH` env var is still recognized — when set
  to a `.jsonl` path, its parent directory becomes the dmn_cycles
  base.
- Nothing in Redis. Nothing on the bus. Nothing in other files.

**Growth caps:** monthly rotation caps cumulative growth but does NOT
cap any single month. A pathological month (extended spiral loop) could
still produce a large file. The spiral-detection logic breaks runaway
cycles before they could plausibly accumulate >GB of events;
secondary size-based rotation within the month is available if that
ever proves insufficient.

---

## Related Docs

- `MEMORY_ARCHITECTURE.md` — the conceptual parent for DMN cycles.
  This subsystem is the instrumentation layer; the memory
  architecture doc describes what the cycles DO.
- `documents/00-meta/ARCHITECTURE.md` — the DMN phase model (`SELF_NARRATIVE`,
  `REFLECTION`, `DIARY_REVIEW`, etc.) is defined there. The
  `phase_advance` event captures transitions between those phases.
- `HEMISPHERE_STATUS_ARCHITECTURE.md` — the `cycle_suspend` and
  `cycle_resume` events are emitted from the DMN's handling of
  hemisphere-outage state. This logger records the transitions; the
  status subsystem provides the facts.
- `NOPE_EVENTS_ARCHITECTURE.md` — a similar append-only JSONL
  subsystem with explicit schema versioning and rotation. The
  `nope_events` design (monthly YYYY-MM rotation, append-only) is
  the pattern this logger now follows.
