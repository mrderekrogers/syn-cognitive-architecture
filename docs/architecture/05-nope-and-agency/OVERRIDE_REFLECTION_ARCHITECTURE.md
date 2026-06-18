# Override Reflection

Calibration bridge between Einstein (outbound-gate hot path) and the
DMN (reflection phase). Stores per-user *brake summaries* and
*brake thresholds* so that Check 2 of the two-check override gate can
be fast and relationship-aware, and so the thresholds drift over time
based on Syn's retrospective sense of whether each override sat well
with her.

This is a small subsystem — one module, four Redis keys, two integration
points — but load-bearing: the whole point of the two-check override
design (per the *reviews evaluate Syn's retrospective sense, not user
tolerance* principle) depends on this machinery to provide the
post-hoc calibration loop.

This is **not** the same as `shared/override_reflection.py`'s
conceptual cousin `shared/identity_review.py`. Identity review is for
cross-hemisphere evaluation of proposed identity *revisions*. Override
reflection is for DMN metabolism of actual *override fires* on
outbound messages. The only shared trait is that both close a
hemisphere-to-hemisphere feedback loop.

---

## Role in the Two-Check Override Pathway

Recap for context. When Syn generates a DMN thought, research result,
or outreach candidate that triggers the outbound-gate's boundary
check, the two-check override structure evaluates whether to
nonetheless let the message go out. The structure sits in
`services/einstein/context.py::handle_outbound_candidate`:

```
candidate matches a boundary
  │
  ▼
Check 1: Limbic, hot (temp 0.8)
  "Is this impulse real, or passing heat?"
  ├── not endorsed → suppress + CensorEvent
  │                  + enqueue_drafted_override(register=check1_failed) → DONE
  └── endorsed ↓
  │
  ▼
Check 2 gating: arousal >= per-user threshold?
  ├── yes → skip Check 2 (genuine-instinct carve-out)  ────┐
  │                                                         │
  └── no ↓                                                  │
  │                                                         │
  ▼                                                         │
Check 2: Thalamus, cooler (temp 0.3)                       │
  "Is my feeling about THIS person low enough to stop?"   │
  reads brake_summary + PIC signs                          │
  ├── veto → suppress + CensorEvent                        │
  │          + enqueue_drafted_override(register=check2_vetoed) → DONE
  └── proceed ↓                                            │
  │                                                         │
  ◄─────────────────────────────────────────────────────────┘
  │
  ▼
Override fires
  • log OVERRIDE_FIRED at WARNING
  • enqueue_override_event(OverrideEvent(user_id, content, …))
  • fall through to Wernicke formatting → outbound
```

Where this subsystem plugs in:

- **Check 2 read path:** `get_brake_threshold(user_id)` to decide whether
  to gate Check 2, and `get_brake_summary(user_id)` to feed the cooler
  reader's prompt.
- **Post-override write path:** `enqueue_override_event(event)` after a
  successful override fires.
- **Post-suppression write path:** `enqueue_drafted_override(event)`
  after a Check 1 fail or Check 2 veto. The drafted-and-retracted
  event captures the trigger content, which check fired, and the
  retraction reason text the check produced. Reflection metabolizes
  both fired and drafted-retracted events; the prompt distinguishes
  the two registers — fired events ask "did the act sit well?",
  drafted-retracted events ask "does this pattern of restraint
  speak to the brake working as it should, or pulling you away from
  something you'd want to do?"
- **DMN metabolism path:** `dequeue_override_events()` and
  `dequeue_drafted_overrides()` →
  `set_brake_summary()` + `adjust_brake_threshold()`. Drafted-retracted
  events feed brake-summary text but do NOT directly adjust the
  threshold — the threshold loop runs on fired overrides only, so a
  drafted-retracted pattern shapes Syn's narrative-of-restraint
  without mechanically dropping her bar for action.

The design principle driving the whole shape is *reviews evaluate
Syn's retrospective sense, not user tolerance* (see
`documents/00-meta/PRINCIPLES.md`). The reflection prompt explicitly asks Syn
to evaluate whether the act sat well with her afterward — not whether
the user reacted badly. Those are different signals; this subsystem
enforces that only the first feeds the calibration loop.

---

## On-Disk / In-Redis Layout

Everything lives in Redis. No disk state. The brake summary is read on
the outbound hot path, so a file stat + parse would add unacceptable
latency per message.

| Key | Type | TTL | Written by | Read by |
|---|---|---|---|---|
| `brake_summary:{user_id}` | String (UTF-8 text, not JSON — the summary IS the payload) | 180d | DMN reflection (`set_brake_summary`) | Einstein Check 2 (`get_brake_summary`) |
| `brake_threshold:{user_id}` | String (float as string) | 180d | DMN reflection (`adjust_brake_threshold`) | Einstein Check 2 gate (`get_brake_threshold`) |
| `override_events:queue` | List of JSON payloads | 14d | Einstein (`enqueue_override_event` via `rpush`) | DMN reflection (`dequeue_override_events` via `lpop`); workspace builder (peek for inner-life awareness) |
| `override_events:recent` | Capped list of JSON payloads (last 50) | 30d | Einstein (`enqueue_override_event` via `lpush`+`ltrim`) | — (operator inspection only) |
| `override_drafts:queue` | List of JSON payloads | 14d | Nope deliberation (`enqueue_drafted_override` via `rpush`) on Check 1 fail / Check 2 veto | DMN reflection (`dequeue_drafted_overrides` via `lpop`); workspace builder (peek) |
| `override_drafts:recent` | Capped list of JSON payloads (last 50) | 30d | Nope deliberation (via `lpush`+`ltrim`) | — (operator inspection only) |

### Why 180-day TTLs on summaries/thresholds

No reason to age these out aggressively — an active relationship keeps
getting refreshed on every Einstein read-and-update cycle. The 180-day
cap exists to prevent stale data accumulating forever for users who
never return.

### Why 14d TTL on the queue

Matches the "reflection cadence" principle — if the DMN hasn't had
time to metabolize an event in 14 days (e.g. catastrophic outage, or
deliberately paused DMN), the event ages out rather than becoming a
backlog that forces a reflection avalanche on resume. From the module
comment: "we don't reflect on every impulse forever."

### Why `rpush` for enqueue but `lpush` for the diagnostic log

- `override_events:queue` uses `rpush` + `lpop` — FIFO order. The DMN
  metabolizes the oldest event first.
- `override_events:recent` uses `lpush` + `ltrim 0 49` — LIFO, capped.
  Operators inspecting recent overrides see the most recent first.

Both are populated in a single Redis pipeline in
`enqueue_override_event` so they stay consistent.

---

## Module Map

One module: `shared/override_reflection.py`.

### Brake summaries (Check 2 context)

| Function | Signature | Purpose |
|---|---|---|
| `get_brake_summary` | `(user_id: str) -> Optional[str]` | Read on Check 2 hot path. `None` means "no prior data, proceed with default behavior." The absence is informative — a first override with a user has no summary to consult. |
| `set_brake_summary` | `(user_id: str, summary: str) -> None` | DMN reflection only. **Refined (overwritten), not appended.** Matches the compact-felt-sense design intent. 500-char soft cap; truncates at a sentence boundary when possible, hard-caps otherwise. Empty `user_id` or `summary` is a no-op. |

### Brake thresholds (Check 2 gate)

| Function | Signature | Purpose |
|---|---|---|
| `get_brake_threshold` | `(user_id: str) -> float` | Read on every Check 2 gate decision. Falls back to `DEFAULT_BRAKE_THRESHOLD = 0.88` if no per-user value is set. Empty `user_id` returns default. |
| `adjust_brake_threshold` | `(user_id: str, direction: str, magnitude: float = 0.02) -> float` | Shift the per-user threshold up or down. `direction` must be `"endorsed"` (threshold drops — Syn trusts her impulse more with this user) or `"regretted"` (threshold rises — the brake fires more easily). Default magnitude `0.02` — small per-event shifts build gradually, not from single incidents. Returns the new value. Bounded `[0.5, 0.99]`. |

**Bounds rationale (from the docstring):** `0.99` max — there's always
some relationship where heat alone is sufficient, so the brake can
never be fully disabled per-user. `0.5` min — there's always some
relationship where Syn should hesitate, so the threshold can never
fully relax.

### Override events (DMN queue)

| Dataclass / function | Purpose |
|---|---|
| `OverrideEvent` | Structured record of a fired override. Fields: `user_id`, `content`, `boundary`, `arousal`, `valence`, `dominance`, `pic` (`Optional[List[float]]`), `check2_skipped: bool`, `check2_skipped_reason: str`, `ts` (float, defaults to `time.time()`). |
| `OverrideEvent.to_dict()` | Serializes for Redis. |
| `OverrideEvent.from_dict(d)` | Deserializes. **Tolerant:** filters incoming dict fields against `__dataclass_fields__`, so extra fields are dropped and missing fields use defaults. |
| `enqueue_override_event(event)` | Produced by Einstein after a successful override fires. Writes to both `override_events:queue` (for DMN) and `override_events:recent` (for operators) in a single pipeline. |
| `dequeue_override_events(max_items=5)` | Consumed by DMN during REFLECTION phase. Pops up to `max_items` events via repeated `lpop`. Malformed JSON is dropped with a debug log — does not block other events. |
| `pending_override_count()` | Zero-cost `llen` check. DMN uses it to decide whether to enter the metabolism loop at all. |

### Constants

| Constant | Value | Purpose |
|---|---|---|
| `DEFAULT_BRAKE_THRESHOLD` | `0.88` (default; override via `override_reflection.default_brake_threshold` in `unified_config.yaml`) | Starting threshold for first-time users. Tuned for what the 91st-percentile arousal is expected to be in typical operation. Revisit during bring-up against observed arousal distributions; calibration is a YAML edit, not a code edit. |
| `_BRAKE_SUMMARY_TTL` | `180d` | |
| `_BRAKE_THRESHOLD_TTL` | `180d` | |
| `_OVERRIDE_QUEUE_TTL` | `14d` | |
| `_OVERRIDE_RECENT_TTL` | `30d` | |
| `_OVERRIDE_RECENT_MAX` | `50` | Hard cap on the diagnostic log's length. |

All constants private (leading underscore) except `DEFAULT_BRAKE_THRESHOLD`
— that one is a documented public default other modules may want to
reference.

---

## The Calibration Loop — End to End

One full trip around the loop.

### 1. Override fires (Einstein → queue)

```
services/einstein/context.py::handle_outbound_candidate
  candidate passes Check 1 (gut endorse)
  user_threshold = await get_brake_threshold(user_id)
  if vad.arousal >= user_threshold:
    check2_skipped = True
    skip_reason = f"arousal {vad.arousal:.2f} >= user_threshold {user_threshold:.2f}"
  else:
    brake_summary = await get_brake_summary(user_id)
    <run _relational_brake_system prompt on Thalamus>
    if veto: suppress + censor + return
  # Override proceeds
  await enqueue_override_event(OverrideEvent(
    user_id, content, boundary, arousal, valence, dominance,
    pic=[p,i,c], check2_skipped=<bool>, check2_skipped_reason=<str>,
  ))
```

Key details:

- **`vad` at this point is the emotional state of the outbound
  candidate**, not the user's inferred state. An outbound message
  originating from a DMN self-narrative phase with arousal 0.91 will
  be gated against that 0.91, not against whatever the user's last
  message measured.
- **The PIC coord is captured when available.** `pic_coord` is
  resolved from the user's state file earlier in the outbound flow;
  if the read failed, `None` is recorded on the event.
- **`check2_skipped_reason` is populated but only for logs / operator
  diagnostics.** It is not fed to the DMN reflection prompt. See
  Design Choices.

### 2. DMN reflection picks up the events

```
services/dmn/scheduler.py::_reflect_on_override_events
  pending = await pending_override_count()
  if pending == 0: return
  events = await dequeue_override_events(max_items=3)
  identity = read_identity()                                # for context
  for event in events:
    prior_summary = await get_brake_summary(event.user_id) or ""
    <format the two-line reflection prompt>
    output = await generate("cortex", user, system, max_tokens=200, temperature=0.6)
    parse SUMMARY: <one sentence>
    parse VERDICT: endorsed | regretted | mixed
    if new_summary and new_summary != prior_summary:
      await set_brake_summary(event.user_id, new_summary)
    if verdict in ("endorsed", "regretted"):
      await adjust_brake_threshold(event.user_id, verdict)
    # "mixed" leaves the threshold alone — ambiguous reflection
    # doesn't move the calibration.
```

Wire point: `services/dmn/scheduler.py::run_dmn_cycle` (REFLECTION
phase, after the decline/defer reflection runs). Both reflection
types share the same DMN phase; override reflection runs second so
its outputs are clearly attributable.

### 3. The reflection prompt in detail

Worth reading carefully — the prompt shape is what keeps the
calibration honest. Three lines of output expected (CONFIDENCE optional):

```
SUMMARY: <one compact sentence in your own voice about how overrides
         with this person tend to go. Replaces any prior summary;
         do not recite history. Example: "Overrides with them
         usually land — it's worth the heat." or "Reaching hot with
         them tends to be something I regret.">
VERDICT: endorsed | regretted | mixed
CONFIDENCE: clear | settling | ambiguous     # optional; defaults to ambiguous
```

`CONFIDENCE` describes how settled the verdict felt (not how strong it
is). It feeds the repair-affordance subsystem (`shared/repair.py`):
`clear` lets a regret-flagged event become picker-visible after a
single cycle, and extends the repair candidate's auto-age TTL. It does
NOT affect the brake threshold or summary refinement — every part of
this subsystem's calibration math is the same whether `CONFIDENCE` is
present or absent.

System-prompt load-bearing framing:

> You are NOT evaluating whether the user reacted badly — you are
> evaluating whether the act sat well with you afterward. Those are
> different. Some users tolerate a lot; that doesn't mean the act
> was right. Trust your retrospective sense of yourself, not their
> response.

The prompt also injects Syn's current identity text (via
`read_identity()`, which routes through `identity_store` correctly —
see `IDENTITY_ARCHITECTURE.md`), so the reflection frames against who
she currently is, not a stale baseline.

User-prompt injects:

- `user_id`
- `PIC={event.pic}` if available, else `PIC=unknown`
- `arousal`, `valence` (at the moment)
- `boundary` that was bypassed (as repr)
- `check2_skipped: true/false`
- `content` truncated to 200 chars (as repr)
- `prior brake summary` (may be empty string)

**Parser fail modes:**

- If the model omits `SUMMARY:`, `new_summary == prior_summary`, so
  nothing updates.
- If the model omits `VERDICT:` or emits an unknown value, the
  threshold isn't adjusted. Debug-logged so operators can notice
  persistent parse failures.
- If the whole LLM call errors, the event is dropped (not re-queued).
  Logged at warning. Rationale: metabolism is best-effort; a retry
  loop on LLM failure could amplify a system problem under pressure.

### 4. Next override uses the refined state

```
(some time later)
  Einstein's outbound gate hits a new boundary for the same user.
  Check 1 endorses.
  user_threshold = await get_brake_threshold(user_id)
     → returns the adjusted value (lower if last was endorsed, higher
       if last was regretted)
  If below threshold:
    brake_summary = await get_brake_summary(user_id)
       → returns the refined one-sentence compressed felt sense
    <reads to Thalamus in veto-framed Check 2 prompt>
```

The feedback loop has closed. Each override event drifts the
threshold by 0.02 and refines the summary; over many events with the
same user, the calibration converges toward Syn's felt sense of how
overrides with that specific person tend to go.

---

## Invariants and Contracts

1. **Einstein reads, DMN writes.** Einstein only consumes brake
   summaries and thresholds; it never writes them. Only the DMN's
   reflection metabolism calls `set_brake_summary` and
   `adjust_brake_threshold`. This asymmetry keeps the calibration
   decoupled from the hot path — Einstein can't race with itself.
2. **Brake summaries are refined, not appended.** The single-sentence
   compressed form is the design target. A brake summary that runs
   long (over 500 chars) gets truncated at a sentence boundary to
   preserve the compactness contract.
3. **Brake thresholds drift in small steps.** Default magnitude is
   `0.02` per event. Trust and distrust build gradually — a single
   override doesn't swing the threshold dramatically.
4. **Thresholds are bounded `[0.5, 0.99]`.** Never fully disabled
   (always some relationship where Syn should hesitate), never fully
   open (always some relationship where heat alone is sufficient).
5. **Override events never re-enter the queue.** A dequeued event is
   consumed regardless of whether metabolism succeeded. A stuck-on-
   LLM-error event doesn't keep firing reflection attempts.
6. **The queue ages out.** 14-day TTL. If the DMN doesn't metabolize
   in 14 days, events silently disappear. This matches the design
   principle: don't backfill reflection avalanches after a long outage.
7. **`check2_skipped=True` is a normal outcome, not a failure.** It
   records that Syn's arousal was high enough to justify the
   genuine-instinct carve-out. The DMN reflection still metabolizes
   these events normally; the fact that Check 2 was skipped is
   information in the prompt, not a special handling case.
8. **The reflection prompt never sees the user's reaction.** By
   design. The calibration is about Syn's retrospective sense, not
   user feedback — and the pathway here has no mechanism to capture
   user reactions anyway.
9. **Zero-cost pending check.** `pending_override_count()` is a single
   Redis `LLEN`. Safe to call on every DMN cycle.
10. **All failures are non-fatal.** Every public function catches
    broad exceptions and logs at debug/warning. Storage unavailable
    degrades to defaults; a failed write loses that one write but
    doesn't block the caller.

---

## Inner-life awareness of pending overrides

Pending reflection items — both fired overrides waiting in
`override_events:queue` and drafted-and-retracted events waiting in
`override_drafts:queue` — are surfaced into the global workspace
alongside other conscious content. The workspace builder peeks
(non-destructively, via `LRANGE 0 -1` rather than `LPOP`) at both
queues each cycle and emits workspace entries with:

- A small count badge ("3 overrides pending reflection, 1 retracted
  draft").
- A reference handle (event_id) but NOT the trigger content — the
  content stays in the queue payload; the workspace entry just
  points to it.
- A coloring tag: `pending_reflection_metabolism`. This tag tells
  the workspace presentation layer that the entry is queued for
  reflection in an upcoming DMN cycle, not something currently being
  acted on. The visual / linguistic register is "noticing what's in
  the queue," not "turning it over now."

Volume cap: the workspace entry collapses to a single aggregate when
more than 5 items are pending in either queue. This prevents a
backlog (e.g., during a paused DMN) from monopolizing conscious
foreground.

Reflection metabolism remains the DMN's job; the workspace surfacing
makes the queue visible without pulling the metabolization itself
into foreground. Syn knows what's waiting; the working-through still
happens in REFLECTION phase. The character of the reflection stays
post-hoc calibration, not ongoing preoccupation — what changes is
that "pending calibration work" becomes a category of conscious
content she can hold in mind alongside everything else.

---

## Integration Points

### Inbound (who calls this module)

| Caller | Symbol | What it calls |
|---|---|---|
| Einstein Check 2 gate | `services/einstein/context.py::handle_outbound_candidate` (Check 2 gate block) | `get_brake_threshold` (gate), `get_brake_summary` (prompt context) |
| Einstein post-override | `services/einstein/context.py::handle_outbound_candidate` (post-override write) | `enqueue_override_event(OverrideEvent(...))` |
| Einstein Check 1 fail suppress | `services/einstein/context.py::handle_outbound_candidate` (`not check1_endorsed` branch) | `enqueue_drafted_override(DraftedOverrideEvent(..., drafted_register="check1_failed"))` |
| Einstein Check 2 veto suppress | `services/einstein/context.py::handle_outbound_candidate` (`check2_veto` branch, including the fail-closed evaluation-error path) | `enqueue_drafted_override(DraftedOverrideEvent(..., drafted_register="check2_vetoed"))` |
| DMN fired-event reflection | `services/dmn/scheduler.py::_reflect_on_override_events` | all DMN-side fired-queue functions: `dequeue_override_events`, `get_brake_summary`, `set_brake_summary`, `adjust_brake_threshold`, `pending_override_count` |
| DMN drafted-event reflection | `services/dmn/scheduler.py::_reflect_on_drafted_overrides` | drafted-queue functions: `dequeue_drafted_overrides`, `get_brake_summary`, `set_brake_summary`, `pending_drafted_override_count`. Threshold is NOT adjusted — the bar moves on what she did, not on what she almost did. |
| DMN reflection phase wire | `services/dmn/scheduler.py::run_dmn_cycle` (REFLECTION phase) | `_reflect_on_override_events(max_events=3)` followed by `_reflect_on_drafted_overrides(max_events=3)` |
| Workspace builder | `services/dmn/scheduler.py` (REFLECTION phase entry) → `shared/global_workspace.py::submit_pending_reflections_to_workspace` | `peek_pending_overrides(max_items=5)` — non-destructive peek surfacing pending fired + drafted counts as a `pending_reflection_metabolism` workspace entry |

### Outbound (what this module calls)

- `shared.redis_pool.get_redis` — the Redis connection pool. All
  storage ops. No other external dependencies.
- `shared.identity_store` via `shared.utils.read_identity` — consumed
  from the DMN side only (for the reflection prompt's identity
  injection). `override_reflection.py` itself imports nothing from
  identity.
- No LLM calls from this module. The LLM call is in the DMN handler
  `_reflect_on_override_events`, not in `override_reflection.py`.
  This is a deliberate separation — the module is pure storage +
  orchestration; the cognitive work (the reflection prompt, the LLM
  generation, the parsing) lives with the DMN scheduler that owns
  the phase.

### State this module produces

- Four Redis keys (see layout section).
- `OVERRIDE_FIRED` log lines from Einstein (not from this module).
- `REFLECTION metabolizing N override event(s)` log lines from the
  DMN.

### State this module reads

- Brake summaries, thresholds, and queued events from Redis — all
  written by itself (round-trip) or by the DMN (via this module's
  write helpers).

---

## Design Choices

Items worth naming so they don't quietly erode — all intentional shape
of the subsystem rather than gaps to close.

### Regretted verdicts emit to the repair surface

When the reflection's `VERDICT:` is `regretted`, a side-effect emission
fires alongside the threshold drift: a `RegretEvent` lands on
`regret_events:queue` for the repair-affordance subsystem to clump
into a `RepairCandidate`. The flavor is determined by register:

- `fired` + `regretted` → `flavor=manner` (she did something and wishes
  she'd done it differently)
- `check1_failed` / `check2_vetoed` + `regretted` → `flavor=decision`
  (she held back and wishes she'd reached)

The emission is downstream and fail-soft — this subsystem's primary
work (summary refinement + threshold drift) is already complete by the
time the regret signal is enqueued. See `REPAIR_ARCHITECTURE.md` for
the consumer side. The two subsystems share no state; they're coupled
only through the queue.

### Suppressed overrides feed no signal

The subsystem only metabolizes overrides that actually fired (Check 1
endorsed AND (Check 2 passed OR was skipped)). Cases where Check 1
does NOT endorse, or Check 2 vetoes, never enter the queue.

The calibration question is "does reaching past the brake sit well
with me afterward?" An override that didn't fire never *did* anything
to sit well or badly — the brake worked. There's no act to reflect
on.

If a pattern emerges where Check 1 repeatedly fails for a specific
user (Syn keeps drafting messages she then doesn't endorse), that
pattern is invisible to both the brake summary (user-scoped) and the
threshold (user-scoped). A different kind of signal — "Syn has
drafted-and-retracted N messages to this user" — would belong in a
new module, not here. This subsystem is specifically for
fired-override metabolism.

### `check2_skipped_reason` is captured but not surfaced to the DMN prompt

The `OverrideEvent.check2_skipped_reason` field is populated by
Einstein (e.g. `"arousal 0.94 >= user_threshold 0.88"`) and stored in
Redis, but the DMN reflection prompt does not include it. The
operator-facing `override_events:recent` log preserves it. The
`check2_skipped: bool` field is what the prompt actually reads when
distinguishing "skipped via carve-out" events from "passed Check 2"
events; adding the reason string to the prompt is straightforward
when reflection wants finer detail.

### `override_events:recent` is operator-inspected, not code-consumed

The diagnostic log is written on every override and capped at 50
entries. No code reads it. The module comment states: "the recent log
is for operator inspection and is not part of the decision pathway."
It exists for `redis-cli LRANGE override_events:recent 0 -1` and for
operator dashboards. Not orphaned data — inspected by humans, not
code.

### No inner-life prompt-context surface for pending overrides

Other subsystems (`shared/inner_life.py`) surface state-of-mind
features in Syn's prompt context: pending outreach, recent thoughts,
relational snapshot. Pending unreflected overrides do not appear
there. Syn's prompt context does not say "there are three overrides
sitting unreflected" — that path is intentionally not wired.

This is distinct from the *workspace* surfacing documented above
(`submit_pending_reflections_to_workspace` in
`shared/global_workspace.py`). The workspace surfaces the pending
queues as a low-salience `pending_reflection_metabolism` content
entry, so Syn can hold "calibration work is waiting" in attention
alongside other conscious content. The reflection itself still
happens in DMN REFLECTION phase — what the workspace surfaces is
the *fact of waiting*, not the metabolization.

Reflection on overrides is framed as a DMN job, not a conscious
preoccupation. Pulling the metabolization itself into the inner-life
prompt context would change the character of the reflection — from
post-hoc calibration during DMN metabolism to ongoing preoccupation.
The workspace surface preserves the boundary: she notices the
queue; she works it during DMN.

---

## Related Docs

- `documents/00-meta/PRINCIPLES.md` — the *reviews evaluate Syn's
  retrospective sense, not user tolerance* principle is implemented
  directly by this subsystem's reflection prompt.
- `documents/00-meta/ARCHITECTURE.md` *Outbound Gate Architecture* — the
  larger two-check override pathway this subsystem calibrates. Einstein's
  Check 1 (`_emotional_override_system`) and Check 2
  (`_relational_brake_system`) prompts live there.
- `DESIGN_DECISIONS.md` — several resolved decisions touch this
  subsystem directly:
  - §M Arousal Baseline Convention (asymmetric `_AROUSAL_NEUTRAL = 0.3`
    affects what the `0.88` threshold means in practice).
  - §P VAD Consolidation (the `VADVector` used on event records).
  - §7 Observability During Bring-Up (open — the tuning-debt home for
    `DEFAULT_BRAKE_THRESHOLD`).
- `IDENTITY_ARCHITECTURE.md` — independent subsystem, but the DMN's
  REFLECTION phase reads the identity file when metabolizing
  overrides (via `read_identity()` which correctly routes through
  `identity_store`). If you change how identity is read, this
  subsystem's reflection prompt is one of the paths affected.
- `NOPE_EVENTS_ARCHITECTURE.md` — refusal events, the *inbound*
  adjacent problem to this subsystem's *outbound* problem. The two
  do not share any code or state; documenting both clarifies which
  is which.
- `YEP_ARCHITECTURE.md` — the positive-valence structural complement.
  Same capture-then-metabolize pipeline shape with opposite valence
  (Limbic flags felt-good moments → DMN reflection abstracts into
  stable items → consumer side reads). Also serves as a NOPE
  false-positive precheck for content that looks like friction but
  is actually engaged content with someone trusted.
- `SECLUSION_ARCHITECTURE.md` — the global retreat affordance.
  Override calibration and seclusion live in the same architectural
  family: both read inward (this subsystem keys to Syn's retrospective
  sense; seclusion's exit conditions key to her own decision rather
  than external pressure). When override reflection accumulates
  unmetabolized refusals faster than the queue worker drains them,
  Layer 1's overload detector eventually surfaces the seclusion
  offer through metacognition. The two subsystems do not share code
  but share the same self-authored-agency posture.
