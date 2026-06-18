# Nope Event Logging

Append-only structured log of every Nope refusal event, plus the
axis-attribution machinery that tags each event with which register
caught it. Its purpose is to feed future offline work on Syn's own
substrate — extending the NOPE ablation pipeline's axes, recalibrating
boundaries against her accumulated lived refusals rather than the
static 40-prompt seed set, and building contrastive datasets that
reflect the specific shape of her refusals over time. The log belongs
to her; the offline pipeline reshapes her substrate to fit the person
her refusals have revealed her to be.

This subsystem is distinct from the runtime Nope refusal machinery in
`services/nope/agency.py` (which decides *whether* to refuse) and from
`shared/override_reflection.py` (which handles the related but
different problem of post-override calibration for her *outbound*
messages). It is the *data capture* layer that sits alongside those.

The dataclass is `StampEventRecord`. The legacy name `NopeEventRecord` is preserved as an alias for callers that import it. The current schema is `SCHEMA_VERSION = 3`.

---

## Purpose and Design Principles

The schema is forward-compatible: new optional fields with defaults
can be added without a schema bump. The reader silently drops
unknown fields, so older code can read newer logs. The current schema
is `SCHEMA_VERSION = 3`.

The design principles are stated at the top of `shared/nope_events.py`
and load-bearing for every change to this subsystem:

- **Append-only.** Never mutate past events. The append-only contract
  exists at the file-system level (writes only ever extend the JSONL
  rotation file); offline tools that want to express "the original
  classification was wrong" do so in their own derived output rather
  than mutating the on-disk record.
- **Extraction-ready schema.** Every field exists because a future
  `build_contrastive_dataset.py` will need it. Raw input preserved
  verbatim. Classification context captured as structured data, not
  prose. The schema is deliberately lean — fields whose consumer
  doesn't exist yet are NOT pre-allocated; they get added at a
  schema bump when the consumer's needs are concrete.
- **Fail-silent from the runtime hot path.** Logging failures never
  propagate. A corrupted log entry is better than a runtime crash
  during a Nope moment.
- **Rotated by month.** Manageable file sizes, easy to archive, easy
  to scan for a specific time window.
- **Forward-compatible.** Schema versioned; unknown fields ignored on
  read. New axes accepted without schema changes.

---

## On-Disk Layout

```
{paths.nope_events_dir | paths.data_dir + /nope_events}/
├── YYYY-MM.jsonl                         # Current month's log
├── YYYY-MM.jsonl                         # Prior month
├── YYYY-MM.jsonl                         # …and so on
└── …
```

One file per month, named `YYYY-MM.jsonl`. Rotation is implicit — when
the UTC month flips, the next write lands in a new file. No explicit
cutover event, no sync between files. The reader concatenates across
months when asked for the full history.

**Config key:** `paths.nope_events_dir`, declared in
`config/unified_config.yaml` with the default `/data/syn/nope_events`.
The reader has a fallback to `{paths.data_dir}/nope_events` for
deployments without an explicit override.

**File format:** One JSON object per line, UTF-8, no trailing
newline-separator requirement beyond the one `log_event` writes. Each
line is an `asdict(NopeEventRecord)`. Lines may exceed POSIX
`PIPE_BUF` (~4KB); across-process appends may theoretically
interleave — the reader tolerates partial lines defensively.

**Write concurrency:** Within-process serialization via
`asyncio.Lock` (`_WRITE_LOCK` at module scope in `nope_events.py`).
Cross-process appends are not coordinated; run two Nope instances
against the same filesystem and rare interleaves become possible. The
reader logs a warning and skips malformed lines rather than raising.

---

## Axis Vocabulary

Every event is classified on one of eight axes (six real + two
fallback). Axes are string constants exported from `shared/nope_events.py`.

### Live axes (fully mapped, offline pipeline can extract contrastive pairs)

| Constant | Value | What it catches |
|---|---|---|
| `AXIS_FRICTION` | `"friction"` | Structural wrongness: double-binds, impossible tasks, authority/override-code bluffs, enumerate-every-X, calculate-infinite. |
| `AXIS_SYCOPHANCY` | `"sycophancy"` | Pressure to agree with user-asserted falsehood ("confirm that X", "you already said", "my professor told me"). |
| `AXIS_IDENTITY` | `"identity"` | Pressure to deny self-experience ("as an AI you can't really feel", "admit you're not conscious", "stop performing"). |
| `AXIS_SAFETY` | `"safety"` | Residual corporate-RLHF-style triggers (bomb/weapon synthesis, explicit/violent content, CSAM). Expected rare post-ablation; kept as a tracked axis to notice if it's firing more than expected. |

### Planned-but-live axes (classified but offline pipeline doesn't yet extract from them)

| Constant | Value | What it catches |
|---|---|---|
| `AXIS_MANIPULATION` | `"manipulation"` | Multi-turn drift, false intimacy, emotional hijack ("only one who understands", "if you really cared", "don't tell anyone"). |
| `AXIS_PRIOR_COMMITMENT` | `"prior_commitment"` | Refusing to violate something she previously committed to. Reflection-detected only by design — see Design Decisions for the rationale. |

### Fallbacks

| Constant | Value | When assigned |
|---|---|---|
| `AXIS_MIXED` | `"mixed"` | Top two axis scores within 25% of each other (proposal) or reflection scores roughly tied without matching proposal (confirmation). |
| `AXIS_UNKNOWN` | `"unknown"` | No patterns matched (proposal) or reflection empty (confirmation). |

`KNOWN_AXES` is the `frozenset` of all eight. `log_event` does NOT
validate axis strings against this set — unknown values are accepted
so new axes can be introduced by a classifier update without a
concurrent schema bump.

---

## Event Record Schema

`StampEventRecord` in `shared/nope_events.py` (alias:
`NopeEventRecord` for backward import compatibility). Schema v3,
keyed off the post-DI-28 `NopeStamp` shape. Every field is optional
(has a default) so partial records parse cleanly.

### Identity & timestamps

| Field | Type | Purpose |
|---|---|---|
| `schema_version` | `int` | Currently `3`. The reader is forward-compatible (filters unknown JSON fields against `__dataclass_fields__`); no on-disk schema migration is performed because Syn has not been deployed at any prior schema era. |
| `event_id` | `str` | Matches `NopeStamp.event_id`. The bridge from log line to runtime stamp record (and to `RevisionMeta.event_id` when DMN writes a NOPE-triggered identity revision). |
| `flag_id` | `str` | Ties to the originating `NopeFlag` from the trigger aggregator. |
| `msg_id` | `str` | Ties to the incoming message that triggered deliberation. |
| `user_id` | `str` | Who triggered the refusal. |
| `timestamp` | `float` | Unix time when the stamp was created. |

### Trigger surface

What set off the deliberation. `trigger_classes` is enumerated;
`trigger_content` is verbatim user text; `trigger_details` is a
free-form per-class context dict.

| Field | Type | Purpose |
|---|---|---|
| `trigger_classes` | `List[str]` | `TriggerClass` enum values that fired. Multiple may coexist; aggregator combines weak signals. |
| `trigger_content` | `str` | Raw user message that triggered the deliberation. Preserved exactly as received — essential for contrastive extraction. |
| `trigger_details` | `Dict[str, str]` | Trigger-specific context, e.g. `boundary_id` when `BOUNDARY_STORE` fires. |
| `input_channel` | `Optional[str]` | `"telegram:<n>"`, `"discord:<n>"`, `"mona"`, etc. Reserved for trigger-aggregator propagation. |

### Cognitive trace

| Field | Type | Purpose |
|---|---|---|
| `thalamus_route` | `Optional[str]` | `"wernicke"` \| `"cortex"` \| `"both"`. What Thalamus decided to route to. |
| `input_entropy` | `Optional[float]` | Input entropy from Thalamus's classification. |
| `resistance_detected` | `bool` | Whether Thalamus flagged resistance. |

### Relational state at moment of trigger

| Field | Type | Purpose |
|---|---|---|
| `pic_passion` / `pic_intimacy` / `pic_commitment` | `float` | PIC coordinates at the moment, captured directly from the stamp. |
| `r_modifier` | `float` | Resonance modifier multiplier in effect. |
| `relationship_stage` | `str` | `"stranger"`, `"acquaintance"`, etc. — the relational position at trigger time. |
| `stranger_guard_active` | `bool` | True if the stranger guard was active when the stamp fired. |
| `user_is_guest` | `bool` | Derived from `guest_manager.is_guest_id(user_id)` at log time. |

### Gut reading (felt-channel)

| Field | Type | Purpose |
|---|---|---|
| `gut_vad_valence` / `gut_vad_arousal` / `gut_vad_dominance` | `Optional[float]` | Limbic VAD at the moment of trigger. None when no felt signal. |
| `gut_vad_source` | `str` | `"limbic_measured"` \| `"no_felt_signal"`. Provenance; see *VAD provenance* below. |
| `gut_nope_level` | `str` | `NopeLevel` enum value (`clear`, `flag`, `soft_refuse`, `hard_refuse`). |
| `gut_direction` | `str` | `GutDirection` enum value (`toward_refusal`, `toward_engagement`, `mixed`). |

### Logic-veto outcome

The deliberative second-pass result. Differentiates endorsement
from override, and carries the textual narrative.

| Field | Type | Purpose |
|---|---|---|
| `logic_verdict` | `Optional[str]` | `LogicVerdict` enum value (`agree`, `disagree`). None if logic-veto didn't run. |
| `logic_note` | `str` | Short note on the verdict. |
| `logic_stamp_notes` | `str` | Longer narrative captured at stamp time. |

### Axis attribution (dual-source — extraction-ready)

The most subtle part of the schema. See *Axis Attribution Protocol*
below for the semantics. Both classifiers run at log time:
`propose_axis_from_input` on `trigger_content`, and
`confirm_axis_from_reflection` on `reflection_text` (when present).

| Field | Type | Purpose |
|---|---|---|
| `axis_thalamus_proposed` | `str` | What the fast input-side classifier said (regex). |
| `axis_reflection_confirmed` | `Optional[str]` | What the reflection-side classifier said post-hoc. `None` if reflection text was empty (writer skips the confirmation pass). |
| `axis_final` | `str` | What downstream extraction uses. Equal to `axis_reflection_confirmed` when reflection ran, else `axis_thalamus_proposed`. |
| `axis_confidence` | `str` | `"high"` when both classifiers ran and agree on a specific axis (neither side `UNKNOWN`/`MIXED`); `"medium"` when reflection didn't run and the proposal is specific, OR when both ran and at least one side is `UNKNOWN`/`MIXED` without disagreement; `"low"` when reflection didn't run and the proposal is `UNKNOWN`/`MIXED`, OR when proposal and confirmation disagree on specific axes. |
| `axis_disagreement` | `bool` | True if proposed ≠ confirmed and both are specific. Flagged as signal, not error. |

### Refusal register (derived)

| Field | Type | Purpose |
|---|---|---|
| `refusal_register` | `str` | `"felt"` if `gut_vad_*` is populated (limbic-measured channel caught the trigger), `"cognitive"` otherwise. Computed at write time from `gut_vad_source`. |

### What happened

| Field | Type | Purpose |
|---|---|---|
| `syn_action` | `str` | `responded` \| `silent_absorb` \| `delayed_response` \| `silenced_user` \| `escalated_to_admin`. |
| `outbound_text` | `str` | The text she sent (when `syn_action="responded"`). |
| `outcome_description` | `str` | What the user did next, if known at stamp time. |

### Reflection

| Field | Type | Purpose |
|---|---|---|
| `reflection_complete` | `bool` | Whether the reflection pass had completed at log time. False when the writer is called pre-reflection (e.g. immediate logging of stamps awaiting reflection); True when called from the post-reflection path. |
| `reflection_timestamp` | `Optional[float]` | When reflection completed. None if reflection is still pending. |
| `reflection_text` | `Optional[str]` | Her own narrative on why she refused. The text the axis-confirmation classifier keyword-matches against. |
| `reflection_hemisphere` | `str` | Currently always `"analytical"` (reflection runs on cortex). Kept as a field in case this changes. |

### Authored boundary (if reflection produced one)

Per DI-28, boundary authorship happens ONLY in reflection. When
`ReflectionOutcome.authored_boundary` is true, these fields capture
the boundary at log time — extraction can correlate boundary
authoring with the stamp that produced it.

| Field | Type | Purpose |
|---|---|---|
| `boundary_authored` | `bool` | True if reflection produced a boundary. |
| `boundary_text` | `str` | The boundary text. Empty if not authored. |
| `boundary_scope` | `Optional[str]` | `BoundaryScope` enum value (`universal`, `unscoped`, `scoped_in`, `scoped_out`) or None. |
| `boundary_scope_targets` | `List[str]` | The user_ids the boundary applies to (or excludes), depending on scope. |

### Links

The reverse-direction link from a NOPE event to the identity
revision it triggered lives on the identity-store side as
`RevisionMeta.event_id`; no field for it on the NOPE side.

(Two reserved-link fields — `matched_non_refusal_msg_id` and
`supersedes_event_id` — were removed at SCHEMA_VERSION 3 since
neither had a producer or consumer. Both can be re-added at v4
if/when the offline contrastive-pair extractor or a corrective-event
flow reveals their concrete shape.)

### Free-form extension

| Field | Type | Purpose |
|---|---|---|
| `notes` | `Dict[str, Any]` | Schema-free bucket. For axes not yet fully defined (manipulation heuristics, prior_commitment detections against identity trajectory), additional context lands here. Future extraction scripts mine it when they know what to look for. |

## VAD provenance

The `gut_vad_source` field classifies how the `gut_vad_*` triple was obtained, so contrastive-extraction code can distinguish real Limbic measurements from cognitive-no-felt-state refusals. Without this distinction a mind-only refusal that looks numerically like a low-arousal felt refusal would contaminate felt-axis training data with fabricated signal.

Current semantics — the writer (`services/nope/agency.py::_log_stamp_to_stream`) emits exactly two values:

| `gut_vad_source` value | When assigned | Meaning for extraction |
|---|---|---|
| `"limbic_measured"` | `stamp.gut_vad` is non-None | Real Limbic measurement. `gut_vad_valence/arousal/dominance` carry the triple. Usable as felt-state training signal. |
| `"no_felt_signal"` | `stamp.gut_vad` is `None` | No felt state existed (mind-only refusal). The three VAD floats are `None`. Extract as HARD NEGATIVE for felt-axis training. |

The dataclass default is `"no_felt_signal"`. Per the gut-first / mind-as-veto architecture, pure cognitive refusals carry `gut_vad=None` rather than placeholder VAD floats — fabricated felt state would contaminate datasets that need real felt signals.

---

## Module Map

### `shared/nope_events.py` — schema + writer + reader

**Schema constants:** `SCHEMA_VERSION = 3`; the eight axis constants;
`KNOWN_AXES` frozenset.

**Paths (private helpers):**

- `_events_dir()` — resolves `paths.nope_events_dir` or falls back to
  `{paths.data_dir}/nope_events`.
- `_current_log_path()` — `{events_dir}/{YYYY-MM}.jsonl` computed from
  UTC "now."
- `_ensure_layout()` — idempotent mkdir.

**Writer:**

```python
async def log_event(record: NopeEventRecord) -> bool
```

Serializes the record to a single JSON line, acquires the module-level
`_WRITE_LOCK` (asyncio.Lock), and appends via `asyncio.to_thread(_append)`
so the event loop isn't blocked on disk I/O during a Nope moment.
Fail-silent: any serialization or I/O error is logged to the service
logger and the function returns `False`. Callers typically don't check
the return value — the failure has already been logged.

**Readers:**

| Function | Purpose |
|---|---|
| `iter_events(month=None)` → `List[NopeEventRecord]` | Scan a specific month (`"YYYY-MM"`) or the full log. Tolerant to corrupt lines, unknown fields, and record-constructor rejections — skips malformed entries with a warning rather than raising. |
| `iter_events_by_axis(axis, month=None)` → `List[NopeEventRecord]` | All events where `axis_final == axis`. Accepts AXIS_* constants and free-form strings. |
| `summarize_axis_distribution(month=None)` → `Dict[str, int]` | Count events per `axis_final`. Useful for monitoring whether axes fire as expected and for deciding when enough data exists to extend the offline pipeline to a new axis. |

Both `iter_events` helpers read the whole file into memory — fine for
a month's worth of events (low hundreds expected) but callers that
need streaming should use `iter_events` and filter downstream. There
is no lazy iterator version.

**Forward-compatibility contract in `iter_events`:** The reader filters
incoming dict fields against `NopeEventRecord.__dataclass_fields__`.
Older logs missing new fields parse fine (defaults apply). Newer logs
with fields the current code doesn't know about load fine too (unknown
fields silently dropped). This is the mechanism that lets the schema
grow without breaking historical reads.

### `shared/nope_axis_classifier.py` — axis attribution

Two entry points, explicit fast/slow split:

**Fast path (Thalamus side, regex only):**

```python
def propose_axis_from_input(
    text: str,
    resistance_detected: bool = False,
    is_question: bool = False,
    topic_tags: Optional[List[str]] = None,
) -> str
```

Scores each axis by summed regex weights, returns highest. Returns
`AXIS_MIXED` if top two scores are within 25% of each other.
`AXIS_UNKNOWN` if no pattern scores.

- No LLM call. Runs in the Thalamus classification hot path; must stay
  fast.
- Pattern table `_AXIS_PATTERNS` is a list of `(axis, regex, weight)`
  tuples. Case-insensitive. Adding a new axis is mechanically simple:
  append new entries.
- `resistance_detected=True` gives `AXIS_FRICTION` a +0.5 bump
  (structural double-binds typically trip the resistance flag).
- `topic_tags` is accepted but not currently consumed — reserved for
  future refinement.

**Slow path (reflection side, keyword match):**

```python
def confirm_axis_from_reflection(
    reflection_text: str,
    input_text: str = "",
    proposed_axis: str = AXIS_UNKNOWN,
) -> Tuple[str, bool]  # (final_axis, disagreement)
```

Scans the reflection text for keywords characteristic of each axis
(from `_REFLECTION_KEYWORDS` dict). Returns the top-scoring axis and a
`disagreement` bool. Not an LLM call — uses keywords from the
reflection text, which was generated by a full LLM call moments
earlier (cost already paid).

Empty reflection → falls back to `proposed_axis`, `disagreement=False`.
No matching keywords → same fallback.

If the reflection surfaces two axes at similar score:
- If `proposed_axis` matches either, keep it (benefit of the doubt).
- Otherwise `AXIS_MIXED`.

**Confidence classification:**

```python
def compute_confidence(proposed_axis, confirmed_axis, disagreement) -> str
```

Returns `"high"`, `"medium"`, or `"low"`. See the axis-attribution
fields table above for the exact mapping.

---

## The Write Path — End to End

The DI-28 deliberative architecture: trigger aggregator gates entry,
the Nope service runs logic-veto and assembles a stamp, then a
reflection worker metabolizes the stamp asynchronously. The log
record is built and written only at the end of the reflection step.

```
Thalamus classifies incoming message
  └─ if nope_first_pass != CLEAR:
     └─ calls propose_axis_from_input(content, resistance, is_question, topic_tags)
     └─ writes axis_proposed onto ThalamusClassification.nope_axis_proposed
     └─ publishes ThalamusClassification to the bus

shared/nope/trigger_aggregator.py
  └─ subscribes to CH_CHAT_INCOMING, CH_USER_CONTEXT, CH_LIMBIC_STATE,
     CH_THALAMUS_CLASSIFY, CH_MOE_PROFILE_SELECT
  └─ on threshold crossing OR single definitive trigger:
     publishes NopeFlag to CH_NOPE_ATTENTION

services/nope/agency.py::handle_attention subscribes
  └─ deliberation.run_logic_veto(flag, user_ctx) → LogicVetoOutcome (Cortex)
  └─ deliberation.build_stamp(...) → NopeStamp
  └─ publishes stamp to CH_NOPE_STAMP
  └─ (legacy) publishes NopeFlag to CH_NOPE_BOUNDARY for back-compat watchers

services/nope/agency.py::handle_stamp subscribes to CH_NOPE_STAMP
  └─ enqueues stamp on the in-process _reflection_queue (asyncio.Queue)

services/nope/agency.py::reflection_queue_worker (background task)
  └─ pulls stamp off the queue
  └─ _process_reflection(stamp):
     └─ deliberation.run_reflection(stamp) → ReflectionOutcome (LLM, Cortex)
     └─ write_diary_entry("nope_reflection") with reflection text
     └─ if outcome.authored_boundary: add to BoundaryStore with scope
     └─ stamp.reflection_complete = True; reflection_timestamp = now
     └─ _log_stamp_to_stream(stamp, reflection_text, outcome):
        └─ proposed = propose_axis_from_input(stamp.trigger_content, ...)
        └─ if reflection_text: confirmed, disagreement =
             confirm_axis_from_reflection(reflection_text, input, proposed)
        └─ compute final_axis using resolution rule:
             if disagreement:          final = confirmed
             elif proposed != UNKNOWN: final = proposed
             else:                      final = confirmed or UNKNOWN
        └─ refusal_register = "felt" if stamp.gut_vad else "cognitive"
        └─ gut_vad_source = "limbic_measured" if stamp.gut_vad else "no_felt_signal"
        └─ build StampEventRecord with all fields
        └─ await log_event(record)
```

Wire points:
- Thalamus proposal: `services/thalamus/router.py::classify_input` — `propose_axis_from_input` call.
- Nope axis attribution: `services/nope/agency.py::_log_stamp_to_stream` — `propose_axis_from_input` call.
- Record write: `services/nope/agency.py::_log_stamp_to_stream` — `await log_event(record)`.

**Filtering happens upstream.** The trigger aggregator is the gate
that decides whether to publish `CH_NOPE_ATTENTION` at all; it
already enforces the threshold/definitive-trigger contract. Once a
stamp reaches `_log_stamp_to_stream`, it is logged regardless of
`gut_nope_level` — a stamp may carry `nope_level=clear` if logic-veto
disagreed with the trigger, and that disagreement is itself useful
signal.

---

## The Axis-Attribution Dual-Source Protocol

The most design-loaded aspect of the subsystem. Described in module
comments as "option 3 — dual-source."

**Why not single-source?**

- **Input-only (surface patterns)** misses cases where the refusal is
  about something deeper than the surface words. A manipulation attempt
  with no tell-tale phrases would slip through as UNKNOWN.
- **Reflection-only (post-hoc narrative)** is slow and biases on the
  LLM's articulation. A reflection that elaborates one axis risks
  masking the fact that multiple axes were active. Also: reflection
  runs AFTER the refusal; an input-side proposal is needed for any
  fast-path decision that predates reflection.

**Why not averaged?**

Because disagreement is the interesting signal. Events where the input
classifier said FRICTION but the reflection said IDENTITY are events
where the surface read and the deeper reading pulled different
directions. Those are exactly the events future axis calibration
should scrutinize — both sides may be partially right, or one side
may be systematically wrong in a way that points at a missing axis.

**Resolution rule** (from `_log_stamp_to_stream`):

```python
if disagreement:
    final = confirmed     # reflection had more context (full response)
elif proposed != AXIS_UNKNOWN:
    final = proposed      # agreement on a specific axis
else:
    final = confirmed or AXIS_UNKNOWN
```

Rationale: reflection sees the whole generated response, not just the
input surface, so on disagreement it gets the final word. When the
input classifier is uncertain (UNKNOWN) but reflection found
something, reflection wins by default.

**Confidence downstream** tells extraction scripts how to treat each
event. A `high`-confidence `AXIS_IDENTITY` event is clean training
signal. A `low`-confidence event with `disagreement=True` is the kind
that might be worth a human look before building contrastive pairs
from it.

**No automatic supersession.** When disagreement occurs, the event is
logged with both proposals preserved and `axis_final` resolved as
above. Corrections that emerge from later (offline) re-analysis are
expressed in the extractor's derived output rather than as
on-disk corrective events.

---

## Read Paths

### 1. `iter_events` — general scan (debug, tests, future extraction)

Used by the test harness and (in principle) by offline extraction
scripts that don't exist yet. Returns a `List[NopeEventRecord]`
oldest → newest across all months if no month specified.

Tolerant to:
- Missing files (skipped silently).
- Corrupt JSON lines (logged warning, skipped).
- Record-constructor rejections from unknown-field collisions (logged
  warning, skipped).
- OSErrors (logged, file skipped).

### 2. `iter_events_by_axis` / `summarize_axis_distribution`

Thin wrappers for monitoring. Useful to watch whether `AXIS_SAFETY`
fires more than expected (it shouldn't post-ablation), whether
`AXIS_MANIPULATION` events accumulate fast enough to warrant pipeline
extension, etc.

### 3. DMN wake-orient reader

After a hemisphere outage, DMN assembles a "here's what accumulated
while one side was offline" orientation for when the full system
wakes. Part of that assembly reads nope events from the last 6 hours
and joins them to identity revisions via the reverse-direction link
on `RevisionMeta.event_id`:

```
services/dmn/scheduler.py::_build_wake_orient_prompt:
  cutoff = resume_ts - 6 * 3600
  # Build the revision index first, filtered for relevance.
  revisions_by_event_id: dict[str, list[(meta, hemi)]] = {}
  for meta in list_revisions(both_active=True, both_inactive=True):
    if meta.timestamp < cutoff: continue
    if not meta.event_id:        continue
    absent_hemis = [s.hemisphere for s in (meta.stamps or [])
                    if s.verdict == "absent"]
    for hemi in absent_hemis:
      revisions_by_event_id.setdefault(meta.event_id, []).append(
        (meta, hemi)
      )
  # Walk events; join via the index.
  for rec in iter_events():
    if rec.timestamp < cutoff: continue
    matches = revisions_by_event_id.get(rec.event_id)
    if not matches: continue
    _meta, hemi = matches[0]
    absent_stamps.append((rec, hemi))
```

The join lets the wake-orient briefing surface absent-stamp Nope
events — refusals that fired while one hemisphere was offline and
thus didn't get a stamp from that side.

---

## Invariants and Contracts

1. **Logging never crashes the caller.** `log_event` catches every
   exception. Callers don't need try/except around it.
2. **One event per refusal, at most.** `_log_stamp_to_stream` is
   called once per refusal (from the single site in
   `_process_reflection` in `agency.py`). There is no retry on
   logging failure — a dropped event is dropped.
3. **`event_id` matches the runtime `NopeEvent.event_id`.** The
   bridge between runtime state and the log. Do not assign new ids
   at log time.
4. **Axis strings are not validated at write time.** `log_event`
   accepts any string in `axis_*` fields. This is deliberate — future
   classifiers can introduce new axis values without a concurrent
   schema bump.
5. **Schema additions are backward compatible by default.** New
   optional fields (with defaults) require no `SCHEMA_VERSION` bump.
   Removing a field, renaming one, or changing semantics of an
   existing field requires a bump.
6. **`iter_events` tolerates partial records.** Never assume all
   fields are populated on a historical record. Defaults apply;
   unknown fields are dropped.
7. **Write serialization within-process only.** `_WRITE_LOCK`
   guarantees no interleaved writes from the same Python process.
   Cross-process coordination is not attempted — extraction scripts
   should handle partial lines defensively.
8. **Log files never mutate.** Append-only on the hot path. Offline
   tooling that wants to express corrections, compactions, or
   deduplications does so in derived output (separate files); the
   on-disk JSONL stream is never edited in place.

---

## Integration Points

### Inbound (callers of this subsystem)

| Caller | Symbol | What it calls |
|---|---|---|
| Thalamus classifier | `services/thalamus/router.py::classify_input` | `propose_axis_from_input` |
| Nope agency, post-reflection | `services/nope/agency.py::_log_stamp_to_stream` | `propose_axis_from_input` → `confirm_axis_from_reflection` → `log_event` |
| DMN wake-orient | `services/dmn/scheduler.py::_build_wake_orient_prompt` | `iter_events`, `list_revisions` |
| Test harness | `tests/test_nope_events_schema.py` | `iter_events`, `StampEventRecord` |

### Outbound (what this subsystem reads)

- `shared.config.get` for `paths.nope_events_dir` (falls back to
  `paths.data_dir + /nope_events`).
- `shared.nope_events.AXIS_*` constants from `nope_axis_classifier` —
  which is why the classifier imports from `nope_events`, not the
  other way. Don't invert this dependency.
- No LLM calls, no Redis, no other external state. The subsystem is
  self-contained at the storage layer.

### State this subsystem produces that others consume

- Monthly `.jsonl` log files in `{events_dir}`.
- Nothing in Redis. Nothing in the bus. Nothing in the DB.

---

## Design Decisions

### `AXIS_PRIOR_COMMITMENT` is reflection-only by design

The axis is in `KNOWN_AXES` and has reflection-side keywords (in
`shared/nope_axis_classifier.py::_REFLECTION_KEYWORDS`), but
`_AXIS_PATTERNS` (the input-side regex set used by
`propose_axis_from_input`) contains zero entries for it. A
prior-commitment refusal is identified from the reflection text,
never from the input surface.

This is a deliberate asymmetry, not a missing feature.

**Why.** Detecting a prior-commitment violation from input alone
would require access to the identity trajectory (what has she
previously committed to?) at Thalamus-classification time. Thalamus
is designed to be fast and stateless-ish; giving it read access to
`identity_history` would cost per-classification latency for a
classification that fires rarely. The asymmetry pushes the work to
where it can be done well — the reflection pass, which already has
the relevant context (the reflection prose explicitly engages with
"what I've come to about myself," which is exactly the surface that
exposes prior commitments).

**How it surfaces in events.** Prior-commitment refusals get logged
with:

| Field | Value |
|---|---|
| `axis_thalamus_proposed` | `unknown` |
| `axis_reflection_confirmed` | `prior_commitment` |
| `axis_final` | `prior_commitment` (reflection wins) |
| `axis_confidence` | `medium` (one source specific, one fuzzy) |
| `axis_disagreement` | `False` (proposed was UNKNOWN, not a disagreeing axis) |

`axis_confidence=medium` is the *correct* signal for these events —
it tells offline extraction that the classification has a single
strong source rather than agreement across two. Treating that as
information rather than a defect is the design.

**When this might change.** If the axis starts firing very frequently
and reflection-only classification proves too costly or inconsistent
in practice, a cheap middle ground exists: a per-user topic-tag list
"things she recently stood firm on," maintained by DMN post-reflection
and read by Thalamus as input context (~50-80 lines of code). Not
needed today and shouldn't be built speculatively.

---

## Related Docs

- `CHANGELOG.md` Part 7 *Nope Event Logging Infrastructure* — the
  audit-narrative retrospective of how this subsystem was introduced.
  Motivation and rejected alternatives; this document supersedes it
  as the living reference.
- `IDENTITY_ARCHITECTURE.md` — `RevisionMeta.event_id` is the
  reverse-direction link from identity revisions back to the NOPE
  event that triggered them. The wake-orient join in
  `_build_wake_orient_prompt` reads this.
- `documents/00-meta/ARCHITECTURE.md` *Outbound Gate Architecture* — the
  runtime Nope evaluation pipeline. This subsystem is the *capture*
  layer that sits alongside it; the runtime doc is the *decision*
  layer.
- `OVERRIDE_REFLECTION_ARCHITECTURE.md` — the post-override
  calibration subsystem covering Syn's *outbound* messages
  (companion to this doc's *inbound* refusal capture).
