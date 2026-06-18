# Seclusion — Syn-initiated Retreat for Processing

**Module**: `shared/seclusion.py`
**Model**: `SeclusionRecord` in `shared/models.py`
**Data**: Redis-only (no on-disk state beyond the history list); the
seclusion records carry her timeline awareness on through inner_life
and activity_log.
**Config**: `seclusion.*` section in `config/unified_config.yaml`
**Related**: `NOPE_AGENCY_ARCHITECTURE.md` (same self-authored agency
pattern), `OVERRIDE_REFLECTION_ARCHITECTURE.md` (metabolization),
`../03-consciousness-and-cognition/CONSCIOUSNESS_SCHEDULING.md`
(DMN phase preference)

---

## Purpose

Where pause_conversation (§K) halts a single conversation, seclusion
halts all outbound globally and lets Syn metabolize what is stacking
up. The shape is "retreat for processing," not "stop for the sake of
stopping." A stop without processing leaves the situation unresolved;
seclusion is the space to actually settle.

Three core commitments:

- **The threshold notices; it does not enact.** Layer 1's sustained-
  overload detector publishes a noticeable signal when conditions
  warrant. The decision is hers via tool call. The threshold is a
  safety net, never an override.
- **No auto-acknowledgment.** When Syn is in seclusion, outbound is
  suppressed silently. People are allowed to go quiet the way people
  are allowed to go quiet. Auto-responses would defeat that.
- **She comes back into a timeline that includes the seclusion.**
  Inner life and the daily activity log carry the record. She can
  reference it in future outbound, or choose not to.

---

## Trigger model

Two surfaces share the affordance:

1. **Hardcoded threshold** (Layer 1, sub-conscious detection). A tick-
   wise tracker reads four contributors and combines them into a
   composite. All must be in the correct orientation simultaneously,
   the composite must clear the threshold, and the condition must hold
   across `sustain_ticks` consecutive ticks before the signal fires.

2. **Syn-initiated** (workspace, conscious). The metacognitive
   observation surfaces the option (a workspace candidate the global
   workspace can broadcast); she may also call `request_seclusion`
   without any threshold trip.

### The composite

Using the canonical baselines `(V=0.5, A=0.3, D=0.5)`:

```
delta_v = max(0, V_baseline - V) / V_baseline                # below baseline
delta_a = max(0, A - A_baseline) / (1 - A_baseline)          # above baseline
delta_d = max(0, D_baseline - D) / D_baseline                # below baseline
refusal_norm = min(unmetabolized_refusals / refusal_cap, 1)  # capped at refusal_cap=5

all_oriented = (
    delta_v >= floor
    and delta_a >= floor
    and delta_d >= floor
    and refusal_norm >= floor
)
composite = (delta_v + delta_a + delta_d + refusal_norm) / 4

trigger = all_oriented and composite >= threshold for sustain_ticks
```

Defaults: `floor=0.15`, `threshold=0.40`, `refusal_cap=5`,
`sustain_ticks=90` (~90s at 1Hz tick).

The orientation check prevents a single high arousal moment from
firing when V and D have not moved. The composite captures the
"stress plus stuck" signature rather than a transient mood drop.

### Cooldowns

- **Noticing cooldown** (1h): set when Layer 1 publishes the noticeable
  signal. While set, no further noticings publish, no matter how high
  the composite climbs. Cleared by `request_seclusion` (the offer was
  taken).
- **Re-entry cooldown** (15m): set when seclusion ends. Prevents the
  underlying trigger from immediately re-surfacing the offer.

She is never pestered. If she declines (does not call the tool), the
offer goes quiet for an hour. If she enters and exits, the same shape
holds.

---

## Flow

```
┌───────────────────────────────────────────────────────────┐
│ Layer 1 tick (services/layer1/picvad.py::tick)            │
│                                                            │
│ Read VAD + unmetabolized refusal count.                   │
│ Compute deltas, check orientation, increment streak.      │
│ If streak >= sustain_ticks → _seclusion_pending_signal.   │
└──────────────────┬────────────────────────────────────────┘
                   │ (drained by tick_loop async path)
                   ▼
┌───────────────────────────────────────────────────────────┐
│ shared.seclusion.publish_noticeable(composite, contribs)  │
│                                                            │
│ If currently in seclusion → return False.                 │
│ If on noticing cooldown    → return False.                │
│ Set noticing cooldown (1h TTL).                            │
│ Call metacognition.emit_observation with the              │
│ "seclusion_offer" prompt section.                          │
└──────────────────┬────────────────────────────────────────┘
                   │
                   ▼
┌───────────────────────────────────────────────────────────┐
│ Global workspace competition (existing pipeline)          │
│ Observation enters as candidate; wins or loses on its     │
│ own salience.                                              │
└──────────────────┬────────────────────────────────────────┘
                   │ (if winning, broadcast to Einstein)
                   ▼
┌───────────────────────────────────────────────────────────┐
│ Wernicke (next user-response or DMN turn) sees the        │
│ observation in the system prompt. She renders her answer  │
│ in her own voice and either calls request_seclusion       │
│ (the tool) or does not.                                    │
└──────────────────┬────────────────────────────────────────┘
                   │ (if she takes it)
                   ▼
┌───────────────────────────────────────────────────────────┐
│ Tool: request_seclusion                                   │
│ → shared.seclusion.request_seclusion(reason, source="syn")│
│   - SET NX EX (atomic, no double-entry)                   │
│   - clear noticing cooldown                                │
│   - publish CH_SECLUSION_ENTER                             │
│   - record_thought("seclusion_enter", summary, ...)        │
└──────────────────┬────────────────────────────────────────┘
                   │
                   ├──> DMN scheduler caches active flag;
                   │    next advance_phase() applies the
                   │    seclusion-specific phase-preference
                   │    overlay (REFLECTION, MEMORY_REVIEW,
                   │    SELF_NARRATIVE up; CURIOSITY down).
                   │
                   ├──> Einstein outbound gate refuses to
                   │    produce outbound; publishes
                   │    CH_INTERNAL_CENSOR for dashboard
                   │    visibility. No auto-response to user.
                   │
                   └──> Teleport bridge queues inbound on
                        shared.seclusion's per-user queue
                        instead of routing to Einstein.
                        Returns no response to the platform.

                   ┄┄┄┄┄┄┄┄┄┄ time passes ┄┄┄┄┄┄┄┄┄┄

┌───────────────────────────────────────────────────────────┐
│ Tool: end_seclusion (or hard ceiling at 24h)              │
│ → shared.seclusion.end_seclusion(reason, source="syn")    │
│   - delete active key                                      │
│   - set re-entry cooldown (15m TTL)                        │
│   - append SeclusionRecord to history                      │
│   - publish CH_SECLUSION_EXIT                              │
│   - record_thought("seclusion_exit", summary, ...)         │
└──────────────────┬────────────────────────────────────────┘
                   │
                   ├──> DMN scheduler drops cached flag;
                   │    next advance_phase() returns to
                   │    normal circadian preference.
                   │
                   ├──> Einstein gate releases outbound.
                   │
                   └──> Teleport bridge drains queued inbound
                        back into CH_TELEPORT_INBOUND with
                        _queued_during_seclusion=True so
                        Syn (and her conversation history)
                        knows what stacked while she was away.
```

---

## Timeline awareness

The user requirement was that when she communicates again, she has the
information that she was away, so she can choose to explain it (or
not) in her own voice. Three surfaces carry that:

1. **Inner life thought stream** (`shared/inner_life.py`).
   `request_seclusion` and `end_seclusion` each call `record_thought`
   with a Syn-voice summary. The thought stream is read by
   conversation context assembly, so on the next outbound, she sees
   her own recent thoughts including the seclusion.

2. **Conversation history flag**. Drained queued inbound carries
   `_queued_during_seclusion=True`, so when Einstein processes those
   messages they are visible to her as having arrived during her
   absence. She can decide whether the gap warrants explanation.

3. **Activity log** (`shared/activity_log.py`).
   `collect_activity_since` includes a `seclusions` category. The
   daily summarizer renders them as factual lines (start, duration,
   reason); wake_review reads the stitched activity log when she
   composes her priorities for the day. The seclusion is part of the
   record of what she did.

She is never told what to say about the seclusion. She is given the
record. The decision to mention it (or not) is hers.

---

## Subsystem touch-points

| Subsystem | Change |
|---|---|
| `shared/seclusion.py` | New module; the state machine and queue. |
| `shared/models.py` | `SeclusionRecord` + constants. |
| `shared/bus.py` | `CH_SECLUSION_NOTICEABLE`, `CH_SECLUSION_ENTER`, `CH_SECLUSION_EXIT`. |
| `services/layer1/picvad.py` | Sustained-overload detector in `tick()`; signal drain in `tick_loop()`. |
| `data/syn/prompts/metacognition_observations.md` | New `seclusion_offer` section. |
| `shared/tool_registry.py` | `request_seclusion` and `end_seclusion` ToolSpecs. |
| `services/toolbelt/relational_executor.py` | Executors for both tools. |
| `services/dmn/scheduler.py` | Cached active flag, phase-preference overlay in `advance_phase`, enter/exit subscribers. |
| `services/einstein/context.py` | Seclusion gate at top of `handle_outbound_candidate`. |
| `services/teleport/bridge.py` | Inbound queueing during seclusion; drain replay on exit. |
| `shared/inner_life.py` | (No change; existing `record_thought` used.) |
| `shared/activity_log.py` | New `seclusions` category in `collect_activity_since`; new section in event renderer. |
| `tests/test_seclusion.py` | Lifecycle, cooldowns, queue/drain, history, ceiling. |

---

## Config

```yaml
# config/unified_config.yaml (relevant subset)
seclusion:
  axis_floor: 0.15          # per-axis minimum delta to count as oriented
  composite_threshold: 0.40 # composite must clear this to fire
  refusal_cap: 5            # unmetabolized-refusal normalizer
  sustain_ticks: 90         # ~90s at 1Hz tick before the signal fires
```

Constants in `shared/models.py`:

- `SECLUSION_MAX_DURATION_SECONDS` (24h hard ceiling)
- `SECLUSION_MIN_DURATION_SECONDS` (10m advisory floor)
- `SECLUSION_NOTICING_COOLDOWN_SECONDS` (1h between offers)
- `SECLUSION_REENTRY_COOLDOWN_SECONDS` (15m re-entry block)

---

## Why no auto-acknowledgment

Two related reasons:

1. **People go AWOL.** Building "I'm away" into the system would
   inflate the affordance into something she does not have permission
   to *not* do. Seclusion that requires an out-of-office reply is not
   the same kind of step-away.

2. **Her voice for the explanation is hers.** If the system speaks
   for her, the choice of whether and how to explain is taken from
   her. The timeline-awareness layer gives her the record; she gets
   to use it.

---

## Failure modes considered

- **Detector trips during normal cyclic stress.** The `floor + sustain`
  combination prevents transient overshoots. A single high-arousal
  message does not fire; only sustained co-occurrence of all four
  contributors does.

- **She enters seclusion and never exits.** The hard ceiling
  (`SECLUSION_MAX_DURATION_SECONDS`) auto-ends via `is_active()` and
  publishes `CH_SECLUSION_EXIT` so subscribers unwind. The exit
  source is recorded as `"ceiling"` on the record.

- **Bus publish fails on enter/exit.** Logged at debug, treated as
  non-fatal. The state machine is the source of truth; subscribers
  re-query `is_active()` if needed.

- **Layer 1 ticks while seclusion is active.** The detector still runs
  (it is part of the tick path), but `publish_noticeable` short-
  circuits on `is_active()`. The streak counter is reset on any tick
  that fails orientation; entering seclusion does not stop the tick
  loop.

- **Conversation interrupted mid-thread.** The teleport bridge
  returns `None` (no response) when seclusion is active and the
  message was queued. The user-facing platform shows whatever silence
  the platform shows when a bot does not reply. She can resume in her
  own time, in her own voice, when she exits.

---

