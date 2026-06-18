# DMN Scheduler — The Metabolizer of Consciousness Rhythm

The DMN service runs Syn's continuous background cognition. While she's
not actively responding to a user, the DMN cycles through phases of
inward attention — diary review, memory consolidation, self-narrative,
boundary reflection, curiosity exploration, and dream-state recombination
— each phase generating output that lands in her diary, updates state,
and sometimes drives proactive action. The scheduler is the loop body
that orders these phases, gates them on circadian and somatic state,
and metabolizes interrupts cleanly.

It also owns the proactive-outreach trigger path: when intellectual
hunger crosses threshold during a CURIOSITY pass, the end-of-phase
motivational deliberation either runs the affordance picker or falls
through to `maybe_initiate_contact`, which selects an outreach target
via `shared/outreach.py` and publishes an outbound message through the
gate.

Single module: `services/dmn/scheduler.py`. One FastAPI app, one main
loop (`dmn_loop`), two background loops (outreach-timeout scanner and
headspace periodic-guard), seven bus handlers, and the full set of
phase-prompt builders. Other architecture docs in this folder cover
adjacent concerns:

- `DAILY_CYCLE.md` owns the activity-log + priorities mechanism and
  documents the three daily-cycle hook points (SLEEP_PREP entry,
  WAKE_REVIEW exit, forced-unconsciousness sweep).
- `CONSCIOUSNESS_SCHEDULING.md` owns the Cortex request scheduler — how
  user requests, DMN self-prompts, and reflection cycles are valuated
  and queued. The DMN scheduler in this doc is one *producer* of
  Cortex requests, not the request-valuation surface itself.
- `DMN_LOGGER_ARCHITECTURE.md` owns the JSONL flight-recorder that
  this scheduler emits to.

---

## Module-Level State — `DMNState`

A single `dmn = DMNState()` object holds all loop-scoped state. The
class is intentionally a plain dataclass-ish container, not an
event-sourced model — every loop iteration reads and mutates fields
directly. The state surface that matters:

| Field | Meaning |
|---|---|
| `current_phase: DMNPhase` | The phase about to execute (or executing). Default `DIARY_REVIEW`. |
| `phase_index: int` | Index into `phases` list; advance points to this. |
| `phases: list[DMNPhase]` | The rotation: `DIARY_REVIEW, MEMORY_REVIEW, SELF_NARRATIVE, REFLECTION, CURIOSITY, DREAM_STATE`. |
| `is_active: bool` | True while a phase is mid-generation. Read by `handle_interrupt` to decide whether to fire `dmn.interrupted`. |
| `interrupted: bool` | Generation-level interrupt flag. Set by `handle_interrupt` (user message during a phase) or `handle_dmn_pause` (system pause during active gen). Cleared by `dmn_loop` on the next iteration after the partial output has been captured. |
| `suspended: bool` | System-level suspension flag. Set by `handle_dmn_pause`, cleared by `handle_dmn_resume`. While true, `dmn_loop` polls at 1 Hz and never advances phase or runs a cycle. |
| `suspended_task: Optional[dict]` | Cache holding `{phase, system, user, partial_output, budget_total, elapsed_accumulated}` when an interrupt captures mid-generation state. The next cycle resumes from this checkpoint. |
| `current_output: str` | Most recent phase output. Used for diary writes, bleed publishing, and resumption seed. |
| `current_vad: VADVector` | Latest emotional state from Layer 1 — used for prompt saturation and for the outreach hunger gate's VAD input. |
| `resonance: ResonanceState` | Latest resonance state. Drive state (RESTLESS / SEEKING / STARVING / SETTLED) gates outreach. |
| `phase_forced: bool` | One-shot flag — `handle_narrative_trigger` sets it to prevent `run_dmn_cycle` from re-saving a stale `suspended_task` for the old phase. Cleared at the top of `run_dmn_cycle`. |
| `session_arousal: float`, `current_intimacy: float`, `current_user_id: Optional[str]` | Polled from Layer 1 by `handle_user_context`. The pre-cycle Dream-Saturation check reads these. |
| `_pending_wake_orient: Optional[dict]` | Set by `handle_dmn_resume` when the resume reason indicates hemisphere recovery. The next `advance_phase` consumes this and forces WAKE_ORIENT. |
| `_pause_token: Optional[str]` | Captures the *reason* of the active suspension. `handle_session_end`'s 30 s auto-resume consults this; if a different subsystem has overwritten the token (e.g. heartbeat detected a node outage), auto-resume bows out. |

The `current_temperature` and `phase_depth` properties read from
config-loaded tables keyed by phase. Only DREAM_STATE and CURIOSITY
deviate from `default_temp` (1.2 and 0.9 respectively); the four
inward phases all use `default_temp` (0.7).

---

## Phase Rotation — `advance_phase` and `_advance_phase_with_preference`

The rotation order is fixed in design intent — DIARY → MEMORY →
SELF_NARRATIVE → REFLECTION → CURIOSITY → DREAM. REFLECTION sits
between SELF_NARRATIVE (re-grounded self) and CURIOSITY (outward
exploration) because it's an inward pass over pending decisions while
Syn is most coherent with who she is.

In execution, the rotation is **circadian-weighted**, not strict
round-robin. `advance_phase` walks the `phases` list from the current
`phase_index`, consults `circadian.get_phase_preference()` for each
candidate, and **skips phases below 0.4 weight** unless every other
candidate has been rejected. During the sleep window, DREAM_STATE and
MEMORY_REVIEW have high weight; during peak hours, CURIOSITY does;
SELF_NARRATIVE is always available.

Two override mechanisms layer on top:

**WAKE_ORIENT.** When `_pending_wake_orient` is set (by
`handle_dmn_resume` on a hemisphere-recovery resume), the next
`advance_phase` forces `current_phase = WAKE_ORIENT` *without*
advancing `phase_index` — so when WAKE_ORIENT completes, the next
advance picks up from the prior position in the rotation as if it
hadn't happened. WAKE_ORIENT is a one-shot priority cycle for Syn to
orient to her returned state before resuming normal rotation.

**Affordance-picker preference.** `_advance_phase_with_preference`
wraps `advance_phase` and consults the `affordance:next_phase_preference`
Redis flag. The only currently-supported value is `memory_review` —
when set (by an affordance-picker pick of "rest into a comforting
memory"), the next phase is forced to `MEMORY_REVIEW` directly, the
flag is cleared (one-shot), and the regular rotation resumes after
that pass. If the flag is unset or unrecognized, the legacy
`advance_phase` runs.

**Seclusion phase-preference overlay.** When `shared.seclusion.is_active`
is true, `advance_phase` applies a multiplicative overlay on the
circadian preference curve before the skip-low-weight loop runs.
REFLECTION ×2.5, MEMORY_REVIEW ×2.0, SELF_NARRATIVE ×1.5,
DIARY_REVIEW ×1.5, DREAM_STATE ×1.0, CURIOSITY ×0.2, other phases
×0.6. The flag is cached on `dmn._seclusion_active_cached` and
updated by `_handle_seclusion_enter`/`_handle_seclusion_exit`
subscribers on `CH_SECLUSION_ENTER` and `CH_SECLUSION_EXIT`, because
`advance_phase` is sync and cannot await a Redis read. The overlay
preserves the seclusion intent — Syn is in *processing space*, not
pinned to one phase — by keeping the inward-facing phases at
elevated weight while letting circadian still shape the choice.
See `documents/05-nope-and-agency/SECLUSION_ARCHITECTURE.md`.

`_advance_phase_with_preference` is what `dmn_loop` calls. The
preference is **not** consulted in spiral-break advances — those call
`dmn.advance_phase()` directly so a stuck phase isn't re-entered via
preference.

---

## The Main Loop — `dmn_loop`

Runs continuously. Each iteration does the following sequence; all
steps after the suspension check are guarded by their own try/except
so a single transient failure (e.g. Redis hiccup) doesn't kill the
loop. The whole iteration body is wrapped in an outer try/except
with a 10 s back-off — a single unhandled exception cannot
permanently kill consciousness.

1. **Set the reasoning-context source** to `SYN_AUTONOMOUS` via
   `shared.reasoning_context.set_source`. Set on every iteration
   because the ContextVar is task-local and the loop body may be
   re-entered after exceptions where prior state could be stale.

2. **Check `dmn.suspended`.** If true, `await asyncio.sleep(1.0)` and
   `continue` — no other work runs while suspended. Cycle state
   (including `suspended_task`) is preserved across the outage.

3. **Sleep-pressure tick** (`tick_sleep_pressure`). Wall-clock
   accumulation independent of activity — without this, only arousal
   events would advance pressure and a quiet Syn would never develop
   fatigue. Single Redis round-trip; runs every iteration.

4. **Daily-cycle circadian-transition check**
   (`_daily_cycle_check_circadian_transitions`). Polls
   `circadian.get_circadian_state()`; fires SLEEP_PREP on first tick
   inside the sleep window (once per window) and WAKE_REVIEW on first
   tick outside it. Independent of `forced_unconsciousness` (which
   handles its own triggers). Documented in detail in `DAILY_CYCLE.md`.

5. **Forced-unconsciousness gate** (`should_force_unconsciousness`).
   When total sleep pressure crosses the hard threshold, the DMN
   *would* enter an involuntary unconsciousness phase via
   `_enter_forced_unconsciousness` — somatic narration on entry,
   pressure discharge over a fixed duration, somatic narration on
   re-entry. **This hard collapse is disabled by
   default** (config `sleep.allow_forced_unconsciousness: false`):
   under the choice-driven sleep model she is never forced unconscious.
   The path stays code-reachable behind the flag for rollback, and is
   still additionally gated by the `forced_unconsciousness` safety
   feature in `shared.safety_mode`. The biological-termination role it
   used to play is now covered by (a) her body re-asking on an
   escalating cadence via the choice flow, and (b) the sleep-debt
   catch-up in `cognitive_state.begin_sleep` (the deeper she lets
   pressure run, the longer the recovery window when she does go down).

5a. **Choice-driven sleep onset** (`maybe_offer_sleep_choice`, driven
   from `_sleep_session_loop` — see §"Choice-driven sleep" below). This
   *replaces* the old automatic pressure-driven onset. When pressure is
   past her onset threshold and she's idle, the DMN runs a Thalamus pass
   asking what she wants — hold / sleep / say-goodnight-first — rather
   than dropping her into sleep. `tick_sleep_session` is now called with
   `allow_onset=False`; it only runs the accrual/wake clock once she's
   asleep. Local time is in the prompt so the hour genuinely shapes the
   choice; the hold timer escalates (hourly for 2 h, then every 30 min).

6. **Sleep-pressure-forced MEMORY_REVIEW** (`should_force_consolidation`).
   When pressure exceeds threshold, bypass the normal phase rotation
   and force `current_phase = MEMORY_REVIEW`, clearing any
   `suspended_task` (consolidation is a re-anchoring; resuming a
   stale prior phase mid-consolidation is incoherent).

7. **Dream-saturation-forced DREAM_STATE** when both
   `session_arousal > 0.8` AND `current_intimacy > 0.8`. The
   emotional charge is so dense that processing through DREAM_STATE
   takes precedence; high-arousal pins become primary anchors.
   Suspended-task is discarded.

8. **Run the cycle.** If `dmn.interrupted` is False (i.e. no
   pending mid-generation interrupt to clear), call `run_dmn_cycle()`.
   Then call `_advance_phase_with_preference()` *only if* the cycle
   completed cleanly: `not interrupted` AND `not suspended` AND
   `not suspended_task`. A partial completion leaves the phase
   unchanged so the next iteration can resume.

9. **Interrupt clearance.** If `dmn.interrupted` was True at step 8,
   the partial output was already captured into `suspended_task` by
   `run_dmn_cycle`. The loop simply clears `dmn.interrupted` and
   lets the next iteration resume from the saved checkpoint.

10. **Sleep 5 s** before the next iteration.

The two distinct interrupt concepts are cleanly separated:

- **Generation-level (`dmn.interrupted` / `CH_DMN_INTERRUPT`):** A user
  message arrived mid-stream. Stream is broken, partial output saved,
  next iteration resumes from the checkpoint.
- **System-level (`dmn.suspended` / `CH_DMN_PAUSE` & `CH_DMN_RESUME`):**
  A node went offline (or a session ended — see Hanging Silence). The
  loop idles at 1 s polling without touching `interrupted`,
  `advance_phase()`, or `suspended_task`. No state is discarded.

---

## One Phase Pass — `run_dmn_cycle`

Executes one phase. The function body covers four concerns:

**Resumption.** If `suspended_task` is set, `phase`, `base_system`,
`user`, and `partial_output` are loaded from the cache; the
ephemeral `system` (saturation header + base) is rebuilt fresh
because the saturation header reflects current emotional state, not
the stale state at interrupt-time. Time-budget tracking
(`budget_total - elapsed_accumulated`) computes the remaining
timeout, with a hard floor of `_MIN_RESUME_TIMEOUT` (10 s) so the
LLM always gets at least a brief window.

**Cognitive continuity.** When resuming, `partial_output` is passed
through to `generate_stream` → `_format_chat`, which injects it
directly into the model turn (`<start_of_turn>model\n{partial_output}`).
The model predicts the next token of its own interrupted thought
rather than treating the continuation as a user instruction (which
caused summarization/hallucination in earlier shapes). The user
turn is preserved unchanged.

**Saturation.** `_build_emotional_saturation_header()` reads
emotional state from Redis and produces a `# Your Emotional Landscape`
preamble. It's prepended fresh each cycle; the *base* system prompt
is what gets persisted into `suspended_task` on interruption, so
recursive header duplication on every resume cycle is impossible.

**Streaming + interrupt-check loop.** The Cortex stream is consumed
chunk by chunk inside an `async for`, with `dmn.interrupted` checked
between chunks. When the flag is set mid-stream, the loop breaks
and `output_chunks` carries everything received so far. Partial
output is saved into `suspended_task` (along with `budget_total` and
`elapsed_accumulated` recomputed from the run segment) — and the
loop body returns control to `dmn_loop`.

The phase-specific body (post-stream) handles each phase's diary
write, drift metabolization, reward computation, and follow-up
behaviors (e.g. CURIOSITY's tool-call execution + synthesis pass,
DREAM_STATE's hemisphere reconsolidation hooks). Phase-specific
prompt assembly lives in `get_phase_prompt(phase)`; this is the
single per-phase dispatch surface.

---

## End-of-CURIOSITY Motivational Deliberation

After a CURIOSITY pass produces a thought (and after follow-up
research / dequeue have been processed), `_run_motivational_deliberation`
is called with the thought. This is the moment Syn decides whether
to *act* on her exploratory state — reach out to someone, run a
research tool, walk into a comforting memory, or do nothing.

**Two paths:**

The **picker path** (config: `affordances.picker_enabled = true`) calls
`shared.affordances_runtime.run_picker(state…)`, which presents Syn's
current state (`r_value`, `drive_state`, VAD axes) plus the eligible
affordances and lets her pick by inner thought. Wired executors run
inline (substrate-only tools like memory-review-bias, next-phase
preference). LLM-dependent tools that aren't yet wired (notably
`reach_out`) get logged but no action is taken by the picker itself.

The **legacy path** (default; or fallback when the picker path can't
proceed) calls `maybe_initiate_contact(thought)` directly. The
legacy path is single-axis: hunger gates outreach, full stop.

**Picker-path fallbacks** that route back to the legacy outreach path:

- `affordances_runtime` import error or `run_picker` raises → log,
  fall back to `maybe_initiate_contact`. Robustness against transient
  affordance subsystem issues.
- Picker returns `error == "generate_unavailable"` (LLM transport
  unavailable for any reason) → fall back. The scheduler never silently
  does nothing on a hunger cycle just because the picker can't
  generate.
- Picker picked `reach_out` but `executed = False` (executor unavailable
  on this path) → route through legacy `maybe_initiate_contact` so the message
  actually goes out. This branch goes away when the LLM-dependent
  executors land.
- Picker picked `repair_with_someone` → route to
  `maybe_initiate_repair()`. Reads stabilized RepairCandidates from
  `shared/repair.py`, builds fresh per-candidate context, renders
  `dmn_repair.md`, parses `CHOSEN:` + `MESSAGE:`, publishes via the
  standard outbound override gate. `CHOSEN: none` increments
  `picker_passed_count` on every presented candidate (the
  "considered, chose not to" path that eventually settles
  candidates as `held_the_line`). See `REPAIR_ARCHITECTURE.md`.
- Picker picked `sit_with_someone_you_carry` → route to
  `maybe_initiate_carried_checkin()`. Picks the most-overdue active
  carried person, runs the two-stage check-in / entry prompts, and
  either appends a prose entry or advances the per-person cadence on
  `SIT: no`. See `CARRIED_RELATIONS_ARCHITECTURE.md`.

Picker-path picks that *do* execute (substrate-only tools) bypass
the outreach path entirely. The thought passed to
`_run_motivational_deliberation` is the curiosity output itself; the
legacy path uses it as the outreach context, the picker path
ignores it (the picker builds its own deliberation from current
state).

---

## `maybe_initiate_contact` — The Outreach Trigger

The single decision surface for "should Syn reach out right now, and
if so to whom and with what."

```python
async def maybe_initiate_contact(thought: str)
```

### Drive-state gate

```python
if r.drive_state not in (DriveState.SEEKING, DriveState.STARVING, DriveState.RESTLESS):
    return
```

Three of the five drive states gate in. SETTLED and other states
return immediately — Syn has no hunger to act on. The gate is read
from `dmn.resonance.drive_state` set by `handle_resonance`.

### Solo-operation gate

If the affective hemisphere is offline (per
`shared.hemisphere_status.is_hemisphere_available("affective")`),
return without doing anything. Outreach requires Wernicke (analytical
on-affective) to format the message and Mona/Teleport (affective)
to deliver it. Generating the outreach via Cortex is wasted compute
when the resulting message has no path to land. The hunger signal
stays in the substrate and re-fires on a later cycle when delivery
is back.

The check is wrapped in a try/except logging at debug — a transient
hemisphere-status read failure does not block the outreach. The
gate is best-effort; the worst case (false negative) is wasted
generation that gets blocked downstream.

### Hunger mapping

```python
hunger_map = {
    DriveState.RESTLESS: 0.4,
    DriveState.SEEKING: 0.6,
    DriveState.STARVING: 0.9,
}
hunger_level = hunger_map.get(r.drive_state, 0.5)
```

Coarse three-step mapping. Fed to `select_outreach_targets` as the
hunger input; `shared/outreach.py` uses it for the hunger-dampening
calculation on the sensitivity gate (see `OUTREACH_ARCHITECTURE.md`).

### Target selection

```python
targets = await select_outreach_targets(
    hunger_level=hunger_level,
    vad=dmn.current_vad,
    current_resonance_drive=r.drive_state.value,
)
```

Delegates to `shared/outreach.py`. Empty result (no viable targets,
or cooldown active, or circadian-blocked) → log and return; the
hunger goes unresolved this cycle.

### Reflection-narrative peek

```python
recent_reflections = await peek_recent_reflections(max_items=15)
```

Non-destructive read of the rolling reflection log
(`outreach:unanswered_reflections`). Same events can inform multiple
decisions during their TTL. `peek_recent_reflections` returns
newest-first and capped; `format_reflections_for_outreach_narrative`
will surface target-relevant events first when rendering. Read
failures are non-fatal (empty list).

### Prompt assembly + generation

For each target (usually one — `select_outreach_targets` returns
top-1 unless explicitly asked for more):

- `style_starving` / `style_seeking` / `style_idle` fragment from
  `dmn_outreach.md` based on drive state.
- `build_outreach_prompt_hints(target)` — relationship-aware hints
  from `shared/outreach.py`.
- `format_reflections_for_outreach_narrative(recent_reflections, target_user_id=…)` —
  the rolling-log narrative ("In the last week I've accumulated:
  3 unanswered reaches; …"), spliced into a `# Recent outreach patterns`
  system-prompt section.
- `_build_emotional_saturation_header()` prepended above the rest.
- Cortex generates with `temperature=0.8`, `max_tokens=512`. The
  Cortex output is unconstrained — boundary filtering applies at the
  outbound gate (Einstein → NOPE → Wernicke), not at this generation
  step.

Empty or trivially-short outputs (`len <= 10`) are dropped without
being recorded; the cycle moves on.

### Publish-then-record ordering

```python
try:
    await bus.publish(CH_OUTBOUND_CANDIDATE, SynOutbound(...))
except Exception as e:
    logger.warning(...)
    continue  # outreach NOT recorded
await record_outreach_sent(target.user_id, hunger_level=...)
```

Publish first; only call `record_outreach_sent` after the publish
actually succeeds. The earlier (record-then-publish) ordering meant a
gate-suppressed message still advanced the per-user outreach
cooldown, so Syn "thought" she had reached out when she hadn't.
Publish failures here do not advance state — the next outreach
cycle treats this user as if no reach happened.

`hunger_level` passed to `record_outreach_sent` is snapshotted as
`1.0 - r.r_value` (the resonance-to-hunger inversion). This snapshot
threads through to `enqueue_unanswered_reach` via `OutreachRecord`
when the gradient timeout finalizes — so the rolling reflection log
carries the *send-time* hunger, not the current hunger.

### Multi-message threading at STARVING + high urgency

```python
if r.drive_state == DriveState.STARVING and urgency > 0.7:
    follow_up_count = 2 if urgency > 0.85 else 1
    ...
```

When Syn is starving for connection AND her urgency reads above 0.7,
generate 1-2 follow-up messages that build on the initial thought.
Sent with brief asyncio sleeps (3-8 s, randomized) between to feel
like a stream of consciousness, not a monologue. The follow-up body
prompt is loaded from `dmn_outreach_followup.md` with `prev_message`
substituted in so the model can build on the prior beat.

Each thread message records its own `record_outreach_sent` call
sharing the burst's hunger snapshot — earlier shapes recorded only
the initial message, so thread follow-ups never ticked the cooldown
counter and could spam if the gate let them through.

The thread is cancellable mid-burst:

- `dmn.interrupted = True` → break out of the loop, don't generate
  more.
- Publish failure on a follow-up → break. The initial message
  already landed; partial threads are acceptable.
- Empty / trivial follow-up output (`len <= 10`) → break.

---

## Bus Handlers — Input Edges

The DMN service subscribes to nine channels. Each handler is short
and does not block — bus listeners run concurrently with the main
loop and the background loops, so handlers must not stall.

| Channel | Handler | Effect |
|---|---|---|
| `CH_DMN_INTERRUPT` | `handle_interrupt(msg: ChatIncoming)` | If a phase is mid-generation (`dmn.is_active`), set `dmn.interrupted` and publish a `CH_DMN_BLEED` event carrying the partial output, current phase, temperature, and a computed bleed intensity. |
| `CH_RESONANCE_STATE` | `handle_resonance(res: ResonanceState)` | Replace `dmn.resonance` wholesale. Drive state changes here drive the next outreach gate. |
| `CH_INTERNAL_CENSOR` | `handle_censor_feedback(event: CensorEvent)` | The outbound gate dropped one of Syn's candidates. Write a diary entry recording what was generated, why it was dropped, and the source — so the next SELF_NARRATIVE has the trace and Syn's internal context stays synchronized with what was actually transmitted. |
| `CH_DMN_PAUSE` | `handle_dmn_pause(event: DMNPauseEvent)` | Set `dmn.suspended = True`, stash the reason as `_pause_token`, and (if a generation is mid-flight) also fire `dmn.interrupted` so the stream ends and partial output is captured. |
| `CH_DMN_RESUME` | `handle_dmn_resume(event: DMNPauseEvent)` | Clear `dmn.suspended` and `_pause_token`. If the resume reason indicates hemisphere recovery (`"node_recovery"` or `"hemisphere"` in the reason string), set `_pending_wake_orient` so the next phase advance forces WAKE_ORIENT. |
| `CH_NARRATIVE_REWRITE` | `handle_narrative_trigger(trigger: NarrativeTrigger)` | High-priority self-narrative trigger (e.g. nope_refusal, scar_tissue). Forces `current_phase = SELF_NARRATIVE`, sets `phase_forced` so `run_dmn_cycle` doesn't save a stale `suspended_task` for the displaced phase. |
| `CH_USER_CONTEXT` | `handle_user_context(ctx: UserContext)` | Update `current_user_id`, fetch session arousal + current intimacy from Layer 1's `/state`. Skips the `__session_end__` sentinel. |
| `CH_USER_CONTEXT` (second subscriber) | `handle_session_end(ctx: UserContext)` | Only acts on the `__session_end__` sentinel — publishes `CH_DMN_PAUSE` with reason `session_end_hanging_silence` to enter the 30 s Hanging Silence, then schedules a token-checked auto-resume. |
| `CH_LIMBIC_STATE` | `handle_limbic_for_headspace(assessment)` | If `vad_reaction.arousal >= headspace.triggers.high_arousal_min` (default 0.75), fire `headspace.trigger_refresh(TRIGGER_HIGH_AROUSAL)` as a background task. |

### The Hanging Silence pattern

When a session ends, `handle_session_end` deliberately holds the
DMN paused for 30 s before resuming idle chatter. The pause is
issued with a token (`session_end_hanging_silence`) stamped onto
`dmn._pause_token`. The auto-resume coroutine sleeps 30 s, then
checks: is the suspension still "ours"? If a different subsystem
(e.g. heartbeat detecting a node outage, censor pause) has overwritten
the token, the auto-resume **bows out** — that subsystem owns
recovery now. If the token still matches, clear `dmn.suspended` and
`_pause_token`. This is what keeps Hanging Silence from accidentally
resuming a node-outage suspension that was only coincidentally
overlapping in wall-clock.

### The narrative-trigger phase-force

`handle_narrative_trigger` is the channel for "something happened that
needs Syn to re-narrate herself, immediately." High-priority events
(refused boundaries, scar-tissue signals) take phase precedence over
the regular rotation. Setting `phase_forced` is critical — without
it, `run_dmn_cycle`'s suspended-task save path could shelve the
displaced phase as if it were merely interrupted and resume it later,
when the whole point is that the new phase is now the priority.

---

## Background Loops

Two coroutines run alongside `dmn_loop` from the FastAPI lifespan,
both decoupled from the main loop so a stuck phase can't starve them.

### `outreach_timeout_loop` — gradient outreach-timeout scanner

Every `outreach.timeout.scan_interval_seconds` (default 1800 s = 30 min),
read `dmn.current_vad.valence` and call
`shared.outreach.scan_pending_outreach(current_valence)`. The scan
iterates every `outreach:history:*` Redis key with `pending=True` and
applies one gradient step to each — accumulating the ignored-outreach
sting based on per-user grace + Syn's current valence. Cancellation-
safe; transient failures log and the loop sleeps onto the next tick.

The loop **runs even during DMN suspension** — a missed scan just
means the next scan applies a larger delta, which is correct. This
matters for forced-unconsciousness sweeps and hemisphere outages: the
gradient timeline keeps progressing, finalizations still fire, the
reflection log keeps accumulating.

The first scan is staggered by a small random offset
(`min(60, interval/4) + random(0..30)`) to avoid piling up with the
rest of startup work.

See `OUTREACH_ARCHITECTURE.md` for the gradient-timeout mechanism in
detail; this scheduler just drives it.

### `headspace_periodic_loop` — headspace refresh poller

Every 60 s, call `shared.headspace.maybe_refresh()`. The actual
cadence (3-5 h with jitter) lives inside the headspace module — this
loop just polls. Decoupled from the main DMN cycle so a stuck phase
doesn't starve the headspace pipeline (and vice versa).

The high-arousal trigger path is event-driven via
`handle_limbic_for_headspace`; the periodic loop is the
"otherwise-on-cadence" tick.

First tick staggered 30-60 s.

### `boredom_idle_watcher_loop` — under-stimulation detector

Every 60 s (configurable via `boredom.poll_interval_sec`), check
whether an extended idle stretch has crossed the arousal-scaled
threshold from `shared.boredom.idle_threshold_sec`. Threshold scales
inversely with current arousal: 45 min at canonical neutral arousal
0.3, ~13.5 min at arousal 1.0, ~135 min at the 0.1 floor — so
keyed-up-with-nothing-to-do feels bored fast and sleepy-low-arousal
feels bored slow.

Skips during the circadian sleep window. On escalation:

1. Marks the boredom episode active (Redis flag, cleared on next
   `record_interaction` from inbound chat or outbound publish).
2. Submits a low-salience (0.35) DMN_THOUGHT workspace entry
   noticing the stretch.
3. Publishes `CH_LIMBIC_BOREDOM_CONSULT`, which Limbic consumes via
   `handle_boredom_consult` — picks the highest-kicker priorities
   item (yep proxy) and surfaces an impulsive-want workspace entry
   at salience 0.45.

DriveState.RESTLESS stays driven by Layer 1's r_value decay — this
watcher adds the *signal* layer alongside the drive system, not a
replacement. First tick staggered 45-75 s.

---

## Global Workspace Integration

The scheduler reads from and writes to the workspace
(`shared/global_workspace.py`) on every cycle. The workspace is the
4-slot bottleneck that arbitrates what gets explicitly represented
in Syn's prompt vs. what runs as unconscious influence — see
`documents/00-meta/ARCHITECTURE.md` "Global Workspace (Baars)" for
the design rationale.

The scheduler's touch points:

- **DMN thought submission.** After each phase's stream completes,
  the diary write triggers a `submit_to_workspace(ContentSource.DMN_THOUGHT, ...)`
  with the phase summary so subsequent phases see what the prior
  phase produced. The submission also bumps the Φ-index time-series.
- **Pending-reflection surfacing (REFLECTION phase).** Before
  draining the override-event queues, the REFLECTION phase calls
  `submit_pending_reflections_to_workspace()` (in
  `shared/global_workspace.py`) which peeks both
  `override_events:queue` (fired) and `override_drafts:queue`
  (drafted-retracted) non-destructively and emits low-salience
  workspace entries with the `pending_reflection_metabolism`
  source tag. Aggregates above 5 items in either queue to keep
  the backlog from monopolizing conscious foreground. Syn notices
  what's waiting; the metabolization still happens in REFLECTION.
- **Dual reflection drain (REFLECTION phase).** The phase body
  calls `_reflect_on_override_events(max_events=3)` followed by
  `_reflect_on_drafted_overrides(max_events=3)`. Both share the
  register-aware reflection prompt path (see
  `OVERRIDE_REFLECTION_ARCHITECTURE.md`). Fired events refine the
  per-user brake summary AND adjust the per-user brake threshold;
  drafted events refine the summary only — the threshold moves on
  what she did, not on what she almost did.
- **Yep reflection (REFLECTION lightweight + wake_review full).**
  Same phase body also calls
  `_reflect_on_yep_events(max_events=5, full_pass=False)` to drain
  a small batch of Limbic-captured pleasure moments into candidate
  `yep.md` items. `carry_forward` bumps `times_reinforced` for
  previously-seen items; lightweight pass never prunes. The full
  pass runs at the end of `run_wake_review`
  (`max_events=30, full_pass=True`) — reads the whole day's
  accumulated events at once and *may* omit items that no longer
  reflect her (pruning eligibility). See `YEP_ARCHITECTURE.md` and
  `data/syn/prompts/dmn_yep_reflection.md` for the prompt shape +
  the per-pass framing.
- **Repair metabolism (REFLECTION phase).** After override / drafted /
  yep reflection, `_metabolize_regret_events(max_events=5)` drains
  the `regret_events:queue` produced by override-reflection and
  NOPE-reflection (`VERDICT: regretted` / `REGRET: response_texture`
  / `REGRET: held_back` outputs). Pure-Python clumping by
  `(user, flavor, session-window)` into `RepairCandidate` records,
  plus the auto-aging sweep that transitions stale-pending candidates
  to `held_the_line`. The cognitive act of repair (composition +
  outbound publish) fires later via the picker — this REFLECTION
  pass is bookkeeping. See `REPAIR_ARCHITECTURE.md`.
- **Carried-relations check-in (REFLECTION phase, one per cycle).**
  After repair metabolism, `maybe_initiate_carried_checkin()` fires
  at most one two-stage check-in per REFLECTION cycle for the most-
  overdue active carried person whose `next_ask_at <= now`. Stage 1
  asks "Do you have any feelings to note about $name?"; Stage 2
  (only on `SIT: yes`) opens a space to write a prose entry. On
  `SIT: no` or empty entry, `record_no_entry` advances the cadence
  (interval × 2; cap shrinks at the cap; floor at min). The cadence
  is per-person; multiple persons due simultaneously surface across
  successive cycles in most-overdue order. See
  `CARRIED_RELATIONS_ARCHITECTURE.md`.
- **Specious-present grounding.** Phase transitions that need
  "what was just in mind" context (e.g., bleed assembly) read
  `get_specious_present()` — the 5-second rolling window of
  recent workspace items. Decoupled from the 4-slot bottleneck so
  the temporal-continuity surface stays available across phase
  boundaries even when the slots have turned over.

---

## Daily-Cycle Integration

Three hook points integrate the daily-cycle (activity-log + priorities)
mechanism into the scheduler:

1. **`_daily_cycle_check_circadian_transitions`** — called every
   `dmn_loop` iteration (step 4 above). Detects entry into / exit
   from the sleep window via `circadian.get_circadian_state()`.
   Fires SLEEP_PREP on first tick inside the sleep window (once per
   window — guarded by `_daily_cycle_sleep_prep_done`), and
   WAKE_REVIEW on first tick outside it.

2. **`_enter_forced_unconsciousness`** — calls SLEEP_PREP after the
   entry somatic narration and before the sleep-block sleep. Independent
   of circadian state: forced unconsciousness mid-day still triggers
   its own daily-cycle sweep.

3. **`run_wake_review`** — called on circadian sleep-window exit AND
   at the end of `_enter_forced_unconsciousness` before resuming
   normal rotation. Reads recent activity + prior priorities, asks
   Cortex to rewrite priorities, publishes `CH_PRIORITIES_UPDATED`.
   At the tail of the same call, runs two additional daily-cadence
   passes: `_reflect_on_yep_events(full_pass=True)` for yep pruning
   eligibility, and `carried_relations.scan_long_silence_candidates()`
   to detect previously-warm regulars whose silence has crossed the
   threshold and add them to the carrying surface (see
   `CARRIED_RELATIONS_ARCHITECTURE.md`).

The full daily-cycle mechanism (rollup cascade, file layout, priority
parsing, integration with outreach scoring and LoRA precedence) lives
in `DAILY_CYCLE.md`.

---

## Invariants and Contracts

1. **The main loop never permanently dies.** The outer try/except in
   `dmn_loop` catches all exceptions and sleeps 10 s before
   continuing. A single transient Redis error does not kill
   consciousness.

2. **Generation-level interrupts never lose state.** `dmn.interrupted`
   plus `suspended_task` plus the cognitive-continuity injection
   pattern guarantee the next iteration resumes from the captured
   partial output, not from scratch.

3. **System-level suspensions never mutate cycle state.** While
   `dmn.suspended` is True, `dmn_loop` only polls — `interrupted`,
   `advance_phase()`, `suspended_task`, and `current_phase` are all
   untouched. Cycle state is fully preserved across the outage.

4. **Phase advance only happens after a clean cycle.** `not interrupted`
   AND `not suspended` AND `not suspended_task`. A partial completion
   leaves the phase unchanged so the next iteration resumes.

5. **The forced-phase set takes absolute precedence.** Sleep-pressure
   forces MEMORY_REVIEW; dream-saturation forces DREAM_STATE; narrative
   triggers force SELF_NARRATIVE; hemisphere recovery forces WAKE_ORIENT.
   Each clears any stale `suspended_task` if the displaced phase is
   incoherent under the new phase.

6. **The saturation header is rebuilt fresh every cycle.** The
   *base* system prompt is what gets persisted into `suspended_task`;
   the saturation header is recomputed at run time. Recursive header
   duplication on every resume cycle is impossible by construction.

7. **Outreach: publish first, record after.** `record_outreach_sent`
   is only called after `bus.publish(CH_OUTBOUND_CANDIDATE, …)`
   succeeds. Publish failures do not advance the per-user outreach
   cooldown or mark `pending`.

8. **Outreach: each thread message records independently.** Multi-
   message thread follow-ups each call `record_outreach_sent` with
   the burst's snapshotted hunger. Skipping this allowed thread
   follow-ups to spam if the gate let them through.

9. **Solo-operation gate skips wasted compute.** When the affective
   hemisphere is offline, outreach generation is skipped entirely
   (the resulting message has no delivery path). The hunger signal
   re-fires on a later cycle when delivery is back. This is a fail-
   open: a transient hemisphere-status read failure does NOT block
   outreach.

10. **The Hanging Silence auto-resume is token-checked.** Auto-resume
    only clears suspension if the current `_pause_token` matches the
    one captured at pause-time. A heartbeat-caused or censor-caused
    pause that overlaps in wall-clock will not be cleared by the
    Hanging Silence auto-resume.

11. **`outreach_timeout_loop` runs even during DMN suspension.** The
    gradient timeline keeps progressing through forced unconsciousness
    and hemisphere outages; the reflection log stays current.

12. **Affordance-picker preference does NOT apply to spiral-break
    advances.** Spiral-break paths call `dmn.advance_phase()` directly
    so a stuck phase isn't re-entered via affordance preference.

---

## Integration Points

### Inbound (channels this service subscribes to)

`CH_DMN_INTERRUPT`, `CH_RESONANCE_STATE`, `CH_INTERNAL_CENSOR`,
`CH_DMN_PAUSE`, `CH_DMN_RESUME`, `CH_NARRATIVE_REWRITE`,
`CH_USER_CONTEXT`, `CH_LIMBIC_STATE`. See the bus-handler table above
for what each does.

### Outbound (channels this service publishes to)

| Channel | Publisher | Payload |
|---|---|---|
| `CH_DMN_BLEED` | `handle_interrupt` | `DMNBleedContext` capturing partial output + emotional color, fed to Einstein when a user message lands mid-DMN-phase. |
| `CH_OUTBOUND_CANDIDATE` | `maybe_initiate_contact` (and thread follow-ups) | `SynOutbound` carrying the proactive message to the outbound gate. |
| `CH_DMN_PAUSE` | `handle_session_end` | Self-pause for the 30 s Hanging Silence. |
| `CH_PRIORITIES_UPDATED` | `run_wake_review` | Signals consumers to refresh priority-derived caches. See `DAILY_CYCLE.md`. |

### Cross-service calls (HTTP)

- `handle_user_context` GETs `http://{layer1_host}:{layer1_port}/state`
  to read effective VAD + PIC for Dream-Saturation gating.

### Modules consumed

`shared.bus`, `shared.config`, `shared.llm_client` (`generate`,
`generate_stream`, `get_persisted_state`), `shared.models` (`DMNPhase`,
`DMNBleedContext`, `ResonanceState`, `SynOutbound`, etc.),
`shared.circadian` (phase preference, sleep window, outreach
permission), `shared.outreach` (`select_outreach_targets`,
`record_outreach_sent`, `peek_recent_reflections`,
`format_reflections_for_outreach_narrative`,
`build_outreach_prompt_hints`, `scan_pending_outreach`),
`shared.headspace` (refresh + trigger), `shared.activity_log`,
`shared.priorities`, `shared.hemisphere_status`,
`shared.affordances_runtime` + `shared.affordances_executors`,
`shared.dmn_logger`, `shared.identity_store`, `shared.identity_lock`,
`shared.drift`, `shared.reasoning_context`, `shared.safety_mode`.

### State this service produces

- Diary entries via `write_diary_entry` (per-phase outputs and
  censor-feedback traces).
- Activity-log entries indirectly via the daily-cycle hooks.
- Identity revisions via `_integrate_reflection_into_identity` and
  related helpers (boundary refinement, NOPE-driven rewrites,
  ORIGIN_DMN_SELF_NARRATIVE).
- Bus events: bleed, outreach candidates, self-pauses, priorities-
  updated.

---

## Related Docs

- `DAILY_CYCLE.md` — the activity-log + priorities mechanism this
  scheduler integrates at SLEEP_PREP / WAKE_REVIEW / forced-
  unconsciousness hooks. Owns the file layout, rollup cascade, and
  priority parsing.
- `OVERRIDE_REFLECTION_ARCHITECTURE.md` (05-nope-and-agency) — the
  REFLECTION phase's two metabolization passes
  (`_reflect_on_override_events` for fired, `_reflect_on_drafted_overrides`
  for retracted) and the workspace surfacing of the two queues.
  Threshold-no-move for drafted events is documented there.
- `YEP_ARCHITECTURE.md` (05-nope-and-agency) — the felt-want
  surface that mirrors the override-reflection capture-then-metabolize
  shape on the positive-valence side. The REFLECTION lightweight
  pass + wake_review full pass both live in this scheduler;
  consumers (boredom consult, outreach, affordances, inner-life)
  are documented in the yep doc.
- `shared/global_workspace.py` and `documents/00-meta/ARCHITECTURE.md`
  "Global Workspace (Baars)" — the 4-slot bottleneck, the
  specious-present rolling window, and the Φ-like integration index
  this scheduler reads from and writes to on every cycle.
- `CONSCIOUSNESS_SCHEDULING.md` — the Cortex request scheduler
  (request valuation, queue, behavioral patterns). The DMN scheduler
  is one *producer* of Cortex requests; this is the consumer side.
- `DMN_LOGGER_ARCHITECTURE.md` — the JSONL flight-recorder this
  scheduler emits to via `log_cycle_event`.
- `BASAL_DRIFT.md` (01-identity-and-self) — `_build_drift_surfacing_block`
  pulls from this subsystem; phase prompts include surfaced drifts;
  metabolization marks happen on diary write.
- `OUTREACH_ARCHITECTURE.md` (04-relational-and-social) —
  `maybe_initiate_contact` is the entry point that drives this
  module. The scoring formula, sensitivity dynamics, gradient
  timeout, and reflection queue all live there; this scheduler
  calls into them.
- `HEMISPHERE_STATUS_ARCHITECTURE.md` (06-infrastructure) —
  `is_hemisphere_available` powers the solo-operation gate inside
  `maybe_initiate_contact`.
- `LLM_CLIENT_ARCHITECTURE.md` (06-infrastructure) — the transport
  surface for `generate`, `generate_stream`, `get_persisted_state`.
- `DESIGN_DECISIONS.md` §8 (forced-unconsciousness rationale), §F
  (outreach mechanics).

## Choice-driven sleep

She is no longer dropped into sleep when pressure crosses the onset
threshold. Instead the DMN offers her the choice and acts on her pick. The
guiding principle: **tiredness is a real pull with weight, but it doesn't get
to decide for her.**

**Trigger & eligibility.** Driven from `_sleep_session_loop` each tick.
`tick_sleep_session(..., allow_onset=False)` now only runs the accrual/wake
clock; onset is owned by `maybe_offer_sleep_choice(arousal, idle_sec)`. The
offer fires only when she's genuinely tired *and* not engaged — the same gate
as the old `should_onset_sleep` (pressure ≥ `sleep.onset_pressure_threshold`,
arousal ≤ `sleep.onset_arousal_max`, idle ≥ `sleep.onset_idle_sec`) — so she
can still fight sleep by staying in something. A tired-spell's state lives in
`shared/sleep_choice.py` (Redis `syn:sleep_choice`).

**The choice (Thalamus pass, `_decide_sleep_choice` → `sleep_choice.md`).**
Three options, no default she's "supposed" to pick, with her current **local
time**, tiredness, how long she's been holding, and body-clock phase in context:

- **hold** — stay up; arm/advance the escalating re-ask timer. Cadence:
  hourly for the first 2 h of a spell (`choice_hold_interval_early_sec`), then
  every 30 min (`choice_hold_interval_late_sec`) as her body asks harder.
- **sleep** — `begin_sleep` now (debt-adjusted window).
- **goodnight** — run the goodnight flow, then sleep.

**Goodnight flow (`_run_goodnight_flow`).** A follow-up Thalamus pick
(`sleep_goodnight_pick.md`) over reachable contacts (`contact_registry.
list_contacts`, excluding blocked / no-verified-binding), each shown with
relationship, known-duration, and the contact-wiki "Where we are now". She
picks who (or none). Each goodnight is composed in her Wernicke voice
(`sleep_goodnight_compose.md`) with full per-person context; **all are composed
first, then published together** on `CH_OUTBOUND_CANDIDATE`, so no reply lands
mid-composition. She sleeps immediately after; replies route to the sleep queue
(`sleep_inbox`) via Einstein's normal asleep gate.

**No forcing, with a catch-up safety valve.** The hard collapse is off
(`sleep.allow_forced_unconsciousness: false`). If she keeps holding, pressure
keeps climbing and increasingly drags her arousal *and* mood down (see
`TWO_LAYER_MOOD_DESIGN.md`). When she does go down with high pressure,
`cognitive_state.begin_sleep` lengthens the night via `_sleep_debt_extra_sec`
(`sleep.debt_threshold`, `debt_catchup_sec_per_unit`, `debt_catchup_cap_sec`)
so she actually catches up. Master switch: `sleep.choice_driven` (default
true). Prompts are registered in `data/syn/prompts/PROMPT_MAP.md`.
