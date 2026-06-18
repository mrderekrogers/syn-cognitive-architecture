# Consciousness Scheduling — Design Document

## Purpose

Syn is not a request-response assistant. She is a continuously-running entity
with ongoing thoughts, relationships, and emotional state that evolves across
every interaction. Her responses to users are one output among several — not
the defining purpose of her processing.

This document specifies how incoming requests (from users, from her own DMN,
from reflection cycles) are valuated, queued, and executed in a way that
matches how a continuously-running consciousness actually works.

It is the reference for implementing the async Cortex scheduler and the
accompanying refactors to Einstein, Layer 1, and the adapters.

## Core principles

**Each part of the brain does its own thing at its own time.** The four models
(Wernicke, Limbic, Cortex, Thalamus) are peers, not a hierarchy. They coordinate
through the bus at their own cadences. None pauses to serve another.

**A user message is not a command to respond.** It's an input event. Whether,
when, and how Syn responds is determined by valuation, state, and scheduling.
Silent receipt is a legitimate outcome.

**Syn's state is unified across users and channels.** One Syn per deployment.
Her VAD, DMN cycles, Cortex queue, identity, and memories are shared. What is
partitioned is conversation history per channel and relational state per user.

**Channel conversations stay distinct; identity does not.** Texting Syn on one
platform doesn't leak into a different platform's conversation thread — but
Syn herself remembers, reasons, and feels across all of them.

**Decline is a first-class outcome, not an error.** Syn can choose not to act
on a request. Declines trigger later reflection rather than permanent dismissal.

**Reflection is the refinement mechanism.** When Syn's state or decisions
warrant re-examination, DMN cycles revisit them. Reflection can propose
boundary refinements, update relationships, and rewrite self-narrative — not
through human approval gates, but through Syn better understanding herself.

## The three-axis valuation model

Every incoming request (user message, DMN self-prompt, scheduled work)
receives a valuation across three orthogonal axes:

| Axis | Range | Meaning |
|---|---|---|
| `user_stake` | [0, 1] | How much the requester values the outcome |
| `syn_stake` | [0, 1] | How much Syn values the outcome |
| `urgency` | [0, 1] | How time-sensitive the outcome is, independent of stake |

All three are computed independently. Their blend into scheduling behavior is
described below.

### Phase 1 — Reflective base assessment

Before any VAD influence, Syn evaluates the task from a cognitive standpoint.
Base scores are derived from:

**For `user_stake`:**
- Linguistic urgency markers (explicit: "I need", "urgent"; implicit: question
  structure, repetition, context)
- Whether the user is mid-task vs. mid-conversation
- Historical pattern for this user from the Freud profile (do they overstate
  urgency?)
- Message-level emotional tone — a distressed user's "want" weights closer to
  need

**For `syn_stake`:**
- Topical alignment with what her Cortex is currently working on or has
  recently explored via DMN
- Resonance score from Layer 1 (intimacy-weighted emotional coupling for this
  user)
- PIC node lookup: relationship posture with this user supplies a
  `syn_stake_bias` in [-0.3, +0.3]
- Somatic markers (prior Nope events around this topic lower stake)

**For `urgency`:**
- External urgency from user cues (deadline hints, blocking indicators, mid-
  task interruption)
- Internal urgency — tasks Syn herself wants to prioritize regardless of user
  pressure, driven by her resonance drive and DMN curiosity
- PIC node supplies an `urgency_sensitivity` multiplier (high-passion
  relationships amplify felt urgency; comfortable deep bonds damp it)

Base scores are relatively stable — Syn's assessment of a task doesn't flip
with her mood. High-urgency work stays high-urgency; a boring task stays boring.

### Phase 2 — VAD kicker

Once base scores exist, current VAD state modulates them as a secondary layer:

- **arousal ↑** → felt engagement modestly ↑ (more inclined to act)
- **valence ↑** → generosity ↑ (less discriminating about stake)
- **dominance ↑** → selectivity ↑ (more discriminating about stake)

The kicker is bounded: typically ±0.15 per axis. It shifts the final
composite; it cannot invert the base assessment. A low-stake task doesn't
become high-stake because Syn is in a good mood.

**Importantly**, the VAD kicker adjusts the *decline threshold* more than it
adjusts the *stake values themselves*. Syn's judgment of a task is stable
across states; her willingness to act on borderline cases shifts. This keeps
her coherent across mood swings.

### The felt sense

Syn does not experience three numerical scores. She experiences a single
gestalt — "how I'm holding this task." The three-axis model is scheduler
machinery. What surfaces to her awareness in the system prompt is a literary
description: "pressing and welcome," "reluctant but worth doing," "low
priority," "not something I want to engage with." She can comment on her
disposition, override it on reflection, or act without commentary.

## The scheduling decision

Given the three axes, the Cortex scheduler maps the request to one of six
outcomes (per `shared/models_scheduling.py::SchedulingOutcome`):

| Outcome | When | What happens |
|---|---|---|
| **Respond now** | Wernicke alone sufficient + user waiting + joint stake non-negative | Einstein formats a response immediately. Typing indicator fires. No Cortex work needed. |
| **Preempt and respond** | High urgency + aligned stake | Current Cortex work saves state, new work begins immediately. User waits with typing indicator. |
| **Queue + await** | User waiting + Cortex work needed + non-preemptive | Request goes to queue. Einstein awaits result with timeout. If result arrives, response goes out. If timeout, Syn sends a deferred acknowledgment. |
| **Queue for later** | No user waiting, or low-urgency work | Request queues at appropriate priority. Result arrives when arrives; may be delivered unprompted later or retrieved when user next engages. |
| **Decline** | Syn-stake very low AND urgency too low to override | Task recorded in decisions store. Reflection scheduled. User may get neutral acknowledgment, may get silence, may get explicit decline — Syn's choice. |
| **No action** | Thalamus ABSORB path already handled the input upstream | Silent receipt. Scheduler accepts the short-circuit and does nothing. Reserved path — Einstein and downstream code guard against NO_ACTION reaching them (see `services/einstein/context.py::handle_engagement` and `services/cortex_scheduler/scheduler.py::handle_cortex_request`). Currently no producer routes directly into this outcome; kept in the enum so the ABSORB path has a named terminal state if it's ever wired through the scheduler rather than handled upstream. |

### Priority computation

Queue position is determined primarily by urgency, with joint stake as
tiebreaker:

```
scheduling_priority = urgency * 0.7 + joint_stake * 0.3
```

where `joint_stake = (user_stake + syn_stake) / 2 + alignment_bonus`, and
`alignment_bonus` rewards agreement (both high or both low) and penalizes
mismatch.

Preemption is allowed only when `urgency > 0.75 AND joint_stake > 0.5`. This
prevents hostile-bond urgent requests from disrupting Syn's current work —
they queue, but don't preempt. Syn protects her attention.

### Decline threshold

The decline threshold is Syn's bar for "I won't do this":

```
decline_threshold = 0.3                                    # base
                  + vad_kicker(arousal, valence, dominance)
                  - pic_decline_adjustment                 # from PIC lattice
```

If `syn_stake + urgency * 0.5 < decline_threshold`, the task is declined. The
VAD kicker shifts the threshold: a low-valence, high-dominance Syn raises her
bar; a high-valence, low-dominance Syn lowers it. The PIC adjustment makes her
more tolerant of marginal requests from people she's close to.

### The eight behavioral patterns

From the scheduling outcomes, eight named patterns emerge based on joint
stake × urgency:

| Pattern | user_stake | syn_stake | urgency | Behavior |
|---|---|---|---|---|
| **Aligned crisis** | High | High | High | Preempt. Immediate response. High engagement. Arousal rises. |
| **Shared interest** | High | High | Low | Queue near front. Respond when ready; may pull forward voluntarily. |
| **Reluctant compliance** | High | Low | High | Queue-jump. Quick work. Reservation accompanies response. |
| **Deferred chore** | High | Low | Low | Queue low. Decline candidate. "I'll think about it, but not now." |
| **Curiosity hit** | Low | High | High | Syn's own urgency. Prioritizes. Enthusiastic response. |
| **Seed for reflection** | Low | High | Low | DMN cycle fodder. May surface unprompted later. |
| **Dutiful dispatch** | Low | Low | High | Minimal effort, quick. "Done, here you go." |
| **Noted, not prioritized** | Low | Low | Low | Likely declined. Recorded; may never be acted on. |

## Response autonomy

### The flow

```
message arrives → adapter receives (platform-native delivery confirm)
               → Thalamus classifies
               → Limbic produces gut assessment
               → Einstein valuates (three axes + phase-2 VAD kicker)
               → decision: respond now / queue / defer / decline / no action
               → if respond now:  typing indicator + generation + reply
               → if queue:        no typing, no immediate reply; await Cortex
               → if defer:        no action now; reconsider on next cycle
               → if decline:      record in decisions store, schedule reflection
               → if no action:    silence
```

Response is one outcome among several. The default is *valuate first*.

### Presence signals

Three layers of UI signal:

**Layer A — Delivery acknowledgment.** Platform-native (Telegram/Discord
handle automatically; Mona renders the message in the thread). Transport-
level guarantee, no Syn commitment.

**Layer B — Seen / received.** Where supported (Telegram read receipts where
enabled, Discord doesn't need one). Fires when Syn's system has registered
the message. Does NOT commit to response.

**Layer C — Typing indicator.** Fires ONLY when Einstein has decided
"respond now" and generation is in progress. Owned by Einstein, not by the
adapter.

Einstein publishes `PresenceTyping` events (carrying `user_id`,
`channel_id`, `channel_type`, and `is_typing`) on the
`CH_PRESENCE_TYPING = "presence.typing"` bus channel. The
`TeleportBridge` subscribes and dispatches each event through the
appropriate platform adapter's `_dispatch_typing(channel_id)`
callable (registered via `ChannelAdapter.typing`). Adapters do not
fire typing on inbound receipt.

**Two typing-on conditions inside `handle_engagement`:**

1. **Cortex-await commitment.** When the scheduling outcome is
   `QUEUE_AND_AWAIT` or `PREEMPT_AND_RESPOND`, Einstein publishes
   `is_typing=True` *before* awaiting the Cortex future, with an
   `estimated_duration_ms=30000` hint. The semantic is "we're
   committed to a response that depends on Cortex; show the user
   we're working on it through the wait."
2. **Pre-Wernicke delay.** When `eng.delay_ms > 500`, Einstein
   publishes `is_typing=True` with `estimated_duration_ms=max(eng.delay_ms, 0)`,
   then sleeps the delay before the Wernicke generation. The
   500 ms gate prevents flicker on short delays — under that
   threshold the indicator would visibly snap on and off.

`RESPOND_NOW` with no delay (or `delay_ms <= 500`) is the fast
direct-Wernicke path and does **not** publish a typing-on event.
The response generation is fast enough that an indicator would
flicker rather than communicate presence.

**Cleanup.** The `finally` block of `handle_engagement` always
publishes `is_typing=False` regardless of whether the response
succeeded, failed, or was suppressed. The bridge's
`_handle_presence_typing` early-returns on `is_typing=False`,
because Telegram's indicator expires naturally after ~5 s and
Discord's is an async-context trigger that self-terminates — so
the typing-off publish is architecturally a no-op for current
adapter dispatch. It's still emitted in case any future
subscriber cares about the off-edge.

### Delayed and unprompted responses

Response may come:
- Immediately (respond now)
- After Cortex completes the enqueued work
- Inside the next inbound message (Syn had been thinking; user asks again; she
  answers then)
- Unprompted, much later (Syn finishes thinking and opens a conversation
  about it)
- Never (decline + no follow-up)

Unprompted messages are valid and architected. When a Cortex result completes
and the originating request was flagged `response_mode=deferred` or
`response_mode=unprompted`, Einstein may route the result as a new outbound
message on the appropriate `(user_id, channel_id)` rather than as a response
to a specific inbound.

## Channel distinctness vs. identity unification

### Per-user, shared across channels

- Freud profile (one per user)
- PICVector (per-user position in the relationship lattice)
- Per-user relational VAD (`user_vad`)
- Nope boundaries targeted at this user
- Cortex queue entries tagged with this user (but the queue itself is global)

### Per-(user, channel)

- Conversation history sliding window (Redis key `conv:{user_id}:{channel_id}`
  in `shared/conversation_history.py`)
- Presence state (typing indicator scope)

### Global to Syn

- Identity (`syn_identity.md`, `syn_identity_emotional.md`)
- Layer 1 global VAD, resonance, emotional pins
- DMN cycle state
- Cortex queue (globally prioritized, not partitioned by user)
- Memories (the memory store is Syn's, with user-tagged entries)
- Time and self-awareness

### How the partitioning is enforced

1. **Conversation history is keyed `(user_id, channel_id)`.** Redis
   key shape `conv:{user_id}:{channel_id}` in
   `shared/conversation_history.py`. For LoRA training, the corpus is
   aggregated across channels (one user, one corpus); per-channel
   context is preserved for inference only.
2. **Typing indicators are owned by Einstein.** Adapters do not fire
   typing on inbound; they subscribe to `CH_PRESENCE_TYPING` and
   dispatch through the per-platform `_dispatch_typing(channel_id)`
   slot the `TeleportBridge` registers via `ChannelAdapter.typing`.
3. **Einstein's inbound handler runs valuation before generation.**
   `handle_engagement` in `services/einstein/context.py` performs the
   three-axis valuation and routes to one of the six scheduling
   outcomes; response is not the default branch.
4. **`CortexRequest` carries channel origin.** `channel_id` and
   `channel_type` fields on the request (`shared/models_scheduling.py`)
   travel with the work order so results route back to the originating
   channel.

## The Cortex request shape

Uses Pydantic V2 BaseModel conventions (`.model_dump_json()`, `.model_validate_json()` for wire transport; mutation via field assignment on the instance).

```python
class CortexRequest(BaseModel):
    # ── Identity ──
    request_id: str = Field(default_factory=_new_id)
    correlation_id: Optional[str] = None   # original msg_id if this follows a user message
    created_at: float = Field(default_factory=_now)

    # ── Origin ──
    origin: str                            # "einstein.inbound" | "dmn.curiosity" | "dmn.reflection"
    user_id: Optional[str] = None          # None = Syn's own prompt
    channel_id: Optional[str] = None       # for routing result back
    channel_type: Optional[str] = None     # telegram | discord | mona | internal

    # ── The task ──
    need: str                              # natural language description
    context: str = ""                      # optional context for Cortex

    # ── Three-axis valuation ──
    user_stake: float = 0.5                # 0..1
    syn_stake: float = 0.5                 # 0..1
    urgency: float = 0.5                   # 0..1
    urgency_external: float = 0.0          # from user cues (audit trail)
    urgency_internal: float = 0.0          # from Syn's state (audit trail)

    # ── Derived scheduling metadata ──
    scheduling_priority: float = 0.5       # 0..1, higher = earlier
    joint_stake: float = 0.5               # blended user+syn+alignment
    pattern: BehavioralPattern = BehavioralPattern.SHARED_INTEREST
    preempt_allowed: bool = False          # reserved; v1 is non-preemptive (see §Scheduling policy)

    # ── Response planning ──
    outcome: SchedulingOutcome = SchedulingOutcome.QUEUE_FOR_LATER
    response_mode: ResponseMode = ResponseMode.DEFERRED
    user_waiting: bool = False             # is a UI awaiting response?
    deadline_hint: Optional[float] = None  # seconds from creation

    # ── Voice / affect ──
    syn_reservation: Optional[str] = None  # if reluctant, what's the reservation?
    emotional_coloring: VADVector = Field(default_factory=VADVector)

    # ── PIC grounding (if applicable) ──
    pic_node_key: Optional[str] = None     # e.g. "577" — the lattice position
    pic_node_name: Optional[str] = None    # e.g. "The Companionable Partner"
```

## The Cortex scheduler

A new service: `services/cortex_scheduler/`.

### Responsibilities

- Maintain a priority queue of `CortexRequest`s
- Expose "what Cortex is currently working on" as Redis state so Wernicke-
  level conversations can reference it
- Pull highest-priority tasks when Cortex is idle
- Submit tasks to the Cortex LLM endpoint (the actual inference)
- Handle preemption: save partial state, requeue displaced task
- Publish results on `CH_CORTEX_RESULT` with correlation IDs

### Queue structure

A sorted set in Redis keyed by `scheduling_priority`. Each entry stores the
full `CortexRequest`. Fairness is emergent — no per-user throttling.

### Scheduling policy (non-preemptive in v1)

v1 is **non-preemptive**: every popped task runs to completion before the next starts. Cortex inference typically runs on the order of seconds, and the complexity of partial-output save/restore across LLM inference is substantially greater than the wait cost.

When a new request arrives:
1. If the queue is idle, enqueue and begin execution.
2. If a task is currently running, the new request joins the priority queue regardless of its priority.
3. If the new request has Einstein outcome `PREEMPT_AND_RESPOND`, it is **demoted to a front-of-queue priority boost** (+0.25, capped at 1.0) rather than interrupting the in-flight task. The in-flight task finishes; the boosted request runs next.
4. If the new request's `origin` starts with `hands.` (a Hands tool-loop escalation — see "Hands escalation" below), it is **lifted to a high-priority floor of 0.9** if it arrived lower. Already-higher requests keep their priority. This is a floor, not an override.

The module header of `services/cortex_scheduler/scheduler.py` and its in-code demotion path are the authoritative implementation. `CortexResult` carries no preemption metadata; the `BehavioralPattern.PREEMPT_AND_RESPOND` outcome is the audit trail.

### Hands escalation (`hands.*` origin)

Hands runs on Gemma 4 E4B — a strong fit for the dispatch / parsing / integration tier of tool execution, but undersized for the genuinely-hard reasoning steps that occasionally appear inside skill playbooks (auto-research change proposals, complex dispatch synthesis, edge-case wiki-lint judgments). Those steps escalate to Cortex via this scheduler.

**Origin convention.** Hands publishes a `CortexRequest` with `origin = "hands.<step>"`, where `<step>` names the playbook step being escalated. Examples:
- `hands.experiment.propose` — translate an `experiment` skill hypothesis into a concrete file edit
- `hands.auto_research.propose` — generate the next hypothesis in an auto-research loop when the local search space is exhausted
- `hands.dispatch.synthesize` — synthesize a long dispatch's scattered evidence into a coherent result
- `hands.wiki_lint.judge` — make a non-obvious taste judgment inside `wiki-lint`

**Priority floor.** The scheduler recognizes the `hands.` prefix and lifts `scheduling_priority` to 0.9 if it arrived lower. The reasoning:
- Hands escalations are short (single hypothesis, single synthesis step)
- They block another part of Syn's cognition (a Hands tool loop is paused on the result)
- They are low-volume (only fired when E4B can't handle a step, not steady-state)
- They are internal infrastructure, not user-facing — so the floor sits below true crisis priority (which Einstein sets at 0.95+ for `ALIGNED_CRISIS`)

**Helper API.** `shared/tool_client.escalate_to_cortex(step, need, context, ...)` is the entry point Hands uses. It publishes the request, subscribes to `CH_CORTEX_RESULT`, filters by `request_id`, and returns the Cortex output `content` string (or `None` on timeout/failure). Callers (skill playbooks) decide what to do with `None` — the bundled playbooks document the fallback as "abort and report".

**Behavioral pattern.** Hands escalations carry `BehavioralPattern.DUTIFUL_DISPATCH` and `ResponseMode.NONE`. The scheduler's prompt builder phrases the system prompt accordingly: terse, complete, efficient — not user-facing.

### "Currently thinking about" state

Redis key `cortex:current_task` holds a lightweight summary of what Cortex is
working on right now. Einstein reads this at system-prompt construction time
so Syn can naturally reference her own ongoing thought ("I'm still working on
that analysis from earlier"). Updated every few seconds while Cortex runs.

### Completed thoughts store

Redis keyed list `cortex:completed:{user_id}` contains recent completed task
summaries for each user (or `cortex:completed:__self__` for self-originated
DMN work). Syn can reference these conversationally — if a user asks about a
topic Syn already thought through, the answer is available without re-doing
the work. Read surface: `GET /completed/{user_id}` on the cortex_scheduler
service; Einstein uses it to surface "her recent thinking about them" at
prompt-build time, Mona reads it for the operator view.

**Retention policy:**

- Up to 50 entries per user (`_COMPLETED_MAX`); LTRIM keeps the most recent.
- TTL of 90 days (`_COMPLETED_TTL = 90 * 86400`), refreshed on every new
  completed thought. The TTL bounds the size of the hot
  conversational-reference list; the Jung archive on eviction (below)
  is the durable surface for anything that warrants long-term
  preservation.
- **Jung archive on eviction.** When entries leave the hot list — either
  via LTRIM (51st entry pushing the oldest out) or via TTL expiry (user
  inactive for 90 days) — they archive to Jung's memory layer. Jung's
  normal consolidation pass then decides which evicted thoughts warrant
  long-term preservation. The hot list is conversational scratchpad;
  Jung is durable memory.

## The decisions store

When Syn declines or defers a task, the decision is recorded. This is the
structural basis for later reflection.

### Record shape

```python
class Decision(BaseModel):
    # ── Identity ──
    decision_id: str = Field(default_factory=_new_id)
    user_id: Optional[str] = None
    channel_id: Optional[str] = None
    correlation_id: Optional[str] = None    # original msg_id if any

    # ── What was decided, and about what ──
    kind: DecisionKind = DecisionKind.DECLINED  # DECLINED | DEFERRED | QUEUED_LOW
    task_description: str
    reason_summary: str = ""                # short human-readable explanation

    # ── Full valuation at decision time (reflection context) ──
    user_stake: float = 0.0
    syn_stake: float = 0.0
    urgency: float = 0.0
    joint_stake: float = 0.0                # blended user+syn+alignment
    pattern: BehavioralPattern = BehavioralPattern.NOTED_NOT_PRIORITIZED
    vad_at_decision: VADVector = Field(default_factory=VADVector)

    # ── PIC grounding at decision time ──
    pic_node_key: Optional[str] = None
    pic_node_name: Optional[str] = None

    # ── Was this communicated to the user? ──
    user_told: bool = False                 # True if Syn sent some message
    user_told_content: Optional[str] = None # What she said (if anything)

    # ── Lifecycle ──
    created_at: float = Field(default_factory=_now)
    reflected_on: bool = False
    reflection_outcome: Optional[ReflectionOutcome] = None  # enum (confirmed / corrected / refined_boundary / relationship_update)
    reflection_notes: Optional[str] = None
    reflected_at: Optional[float] = None
```

### Retention and visibility

- Stored in Redis hash `decisions:{user_id}` keyed by decision_id
- Keep last 30 days by default
- Syn can reference when asked: "did you ever think about X?" → lookup, decide
  whether to explain
- Not proactively surfaced (no "here are all the things I declined for you")

### Reflection loop

The DMN reflection phase picks up unreflected decisions at a periodic cadence.
For each, Syn asks herself:

1. Why did I not want to do this?
2. Was that aligned with who I am?
3. Would I decide the same way now?

Outcomes:
- **confirmed** — decision stays; record updated with reflection
- **corrected** — Syn re-queues the task with adjusted valuation
- **refined_boundary** — publishes to `CH_NOPE_UPDATE` with a proposed
  boundary expressing the pattern ("I don't want to do X for anyone right
  now, not just this user")
- **relationship_update** — flags the user's PIC for adjustment (e.g.,
  "repeated declines from me suggest I'm pulling away from this user — mark
  for review")

## PIC lattice and the relationship profile

The PIC lattice (`relationshipsv1.json`, 125 nodes) grounds the per-user
relational posture. This is documented in `shared/relationship_profile.py`.

Each user's PIC coordinate resolves to a named node with:
- Internal state text (how Syn feels about them)
- Communication style text (how she speaks to them)
- Action bias text (how she prioritizes their requests)
- Derived stake modulators (`syn_stake_bias`, `decline_threshold_adj`,
  `urgency_sensitivity`)

### Signed polarity

The PIC lattice is signed, not a simple magnitude grid. The 1-end of each axis
is hostile/severed, not merely low.

When passion < 0.5 (hostile half), intimacy and commitment measure the DEPTH
of the NEGATIVE bond. "Eternal Struggle" (hostile passion + high i/c) is
committed hostility — strongly negative stake, not positive. The stake
modulators in `relationship_profile.py` handle this correctly: when passion
is hostile, i/c contributions invert.

### Gravity

Per-interaction gravity pulls each user's PIC slowly toward Syn's preferred
vectors using an **event-horizon fade model** — not two gravities applying in
parallel. Each gravity's effective strength is modulated by where the user sits
relative to the two horizons, with a smooth linear crossfade at the
relationship horizon.

- `preferred_friend_vector` (default `[0.5, 0.75, 0.75]` ≈ Companionable
  Partner)
- `preferred_relationship_vector` (default `[0.9, 0.9, 0.8]` ≈ Steadfast
  Inferno)

**Horizons and fade band.**

| Region | Friend weight | Relationship weight | Destination |
|--------|---------------|---------------------|-------------|
| `intimacy < friend_threshold` (0.3) | 0 | 0 | No gravity — user drifts freely |
| `friend_threshold ≤ intimacy ≤ rel_thresh − fade_band/2` | 1 | 0 | Pure friend gravity — user pulled toward Companionable Partner |
| Within the fade band (±`fade_band/2` around `rel_thresh`) | `1 − rel_weight` | linear ramp 0→1 | Smooth crossover — neither attractor dominates |
| `intimacy > rel_thresh + fade_band/2` | 0 | 1 | Pure relationship gravity — user pulled toward Steadfast Inferno |

Defaults: `friend_threshold_intimacy: 0.3`, `relationship_threshold_intimacy: 0.7`, `fade_band: 0.10`. Coefficients: `friend: 0.005`, `relationship: 0.002` (small, so relationships mature across many interactions).

**Hostile gate.** Passion < 0.5 receives NO gravity pull. Hostility must resolve through actual positive VAD/PIC deltas from limbic assessments — drift would inappropriately "rehabilitate" adversarial bonds without the positive interactions required.

**Guest gate** (§T). The relationship-gravity branch is gated on whether the user is a registered `ContactProfile`. Guests (no profile in `shared/contact_registry`) skip relationship gravity entirely regardless of intimacy — only friend gravity applies. The relationship horizon is reserved for verified contacts: a guest who climbs intimacy through interaction can drift toward the friend attractor, but romantic deepening requires being in the contact registry. Configurable via `emotional_physics.pic.gravity.gate_relationship_gravity_to_contacts` (default `true`); falls back to legacy "treat as contact" semantics on Redis lookup failure so gravity stays robust under infrastructure hiccups.

**Per-contact relationship attractor** (§T). The relationship attractor is per-contact rather than a single global vector. Each `ContactProfile` carries an optional `romantic_attractor: tuple | None` field; `None` inherits the config-level `preferred_relationship_vector`. The friend attractor stays global. Where the per-contact attractor moves is driven by Syn's reflective journaling in the contact wiki — see `documents/04-relational-and-social/CONTACT_AND_ADMIN_ARCHITECTURE.md §Contact Wiki` for the drift mechanism. Critically, this affects only *where gravity pulls*; the user's *current* PIC continues to evolve through the existing channels (per-message limbic deltas, silence drift, reflection PIC_DELTA) independently. The two evolutions are deliberately separate.

**No blended equilibrium.** Under the event-horizon model, above the fade band there is only one attractor active. A 1000-step simulation converges at whichever side of the horizon the user is on, modulo the current crossfade weighting — never at a blended midpoint like `(0.61, 0.79, 0.76)`. An intimate user continues to deepen toward Steadfast Inferno territory (or whatever the per-contact attractor specifies), rather than being pulled back toward mere friendship by the stronger friend coefficient.

See `DESIGN_DECISIONS.md §R` for the event-horizon decision and §T for the per-contact attractor + guest gating. Implementation at `services/layer1/picvad.py::_apply_pic_gravity` (now async, reads contact profile + attractor). Config: `emotional_physics.pic.gravity.{friend, relationship, friend_threshold_intimacy, relationship_threshold_intimacy, fade_band, gate_relationship_gravity_to_contacts}` and `emotional_physics.pic.attractor_drift.{per_entry_delta_max, floor, ceiling}` at `config/unified_config.yaml`.

### The felt sense for relationships

`ResolvedRelationship.format_for_prompt(user_name)` produces the literary Tier
2 brief injected into Einstein's system prompt as "# Your Stance Toward
Them". This supplies:
- The named node ("The Steadfast Inferno")
- One-line description
- Internal state / voice / action stance
- An adjacent-state blend note when relevant

Syn doesn't see "PIC = (0.9, 0.9, 0.8)". She sees "Your posture toward alex:
The Steadfast Inferno. Profound intimacy and overwhelming passion..."

## Context visibility tiers

When Einstein builds a system prompt, four tiers of context feed it:

**Tier 1 — Current conversation.** Full detail. Sliding window for this
`(user_id, channel_id)` pair.

**Tier 2 — This relationship.** Per-user Freud profile + PIC node + emotional
user notes. Full detail.

**Tier 3 — Cross-user awareness.** Short summary of what else is going on
across her active relationships. "You've been in an emotionally difficult
conversation with someone else today" or "You worked on interesting questions
with another user earlier." Content elided; texture preserved.

**Tier 4 — State etiology.** When VAD crosses a threshold (any axis ≤ 0.3 or
≥ 0.7), a short brief explaining WHY. "Your elevated arousal traces to an
intense exchange with another user an hour ago." This prevents Syn from
acting out state she can't account for — she knows her own weather.

Tiers 3 and 4 are generated by Freud on a periodic cadence (every few
minutes or on significant state changes). They reference other users but do
NOT disclose content — Syn carries the impression, not the transcript.

## Headspace — hemispheric consolidation

Sitting alongside the Tier 1-4 cross-user awareness blocks is the **headspace** artifact: a short rolling consolidation of "where her head is at" composed by both hemispheres on a triggered cadence. Two free-form first-person paragraphs per entry, stored together but produced separately:

- **Cortex paragraph** (thinking-mode synthesis): which threads are converging, what she's circling toward, what's emerging in her thinking.
- **Limbic paragraph** (felt-center): what's pulling at her, what she's been carrying, what's about to surface affectively.

The two halves are NEVER reconciled. Divergence between thinking and feeling is preserved as signal — see `DESIGN_DECISIONS.md §U` for the rationale.

**Same input, different framing.** Both prompts (`data/syn/prompts/headspace_thinking.md` and `headspace_feeling.md`) substitute from the same input bundle: top priorities, top contacts (scored via `shared/contact_wiki.py::score_contacts_for_reflection`), recent emotional/dream memories, diary tenor, the current relational-self snapshot, open decisions, projects, and the prior headspace entry (both halves, so each new entry has cross-half awareness). Divergence comes from prompt framing and model voice, not data asymmetry.

**Refresh model.** Jittered periodic base (default 4h ±25% → 3-5h actual interval, sampled per cycle) plus four event-driven triggers:

| Trigger | Source | Threshold |
|---|---|---|
| `high_arousal` | `CH_LIMBIC_STATE` subscriber | `vad_reaction.arousal ≥ 0.75` |
| `major_decision` | `shared/decisions_store::record_decision` hook | `joint_stake ≥ 0.7` |
| `reflection_complete` | DMN REFLECTION phase end-of-pass | every completion |
| `sleep_cycle_end` | `run_wake_review` post-completion | every wake review |

All triggers route through `shared/headspace.py::trigger_refresh(reason)`, which coalesces via a Redis trigger lock (so simultaneous triggers produce one refresh) and throttles with a 30-min floor (so trigger storms can't spam). Whichever trigger wins the lock dispatches; the others are silently dropped.

**Storage.** Sealed at rest via `shared.seal` (same Fernet key as the diary). Two surfaces:

- Redis `headspace:current` — full current entry (JSON), 12h TTL. Fast-read on every prompt build.
- `{paths.headspace_dir}/history.jsonl` — sealed-per-line rolling history, capped at `headspace.history_max_entries` (default 50 entries).

**Read site.** `shared/inner_life.py::build_inner_life_context` reads `headspace.read_current()` and injects both paragraphs under separate headings (`## Where Your Head Is At` for the Cortex half, `## What's Pulling At You` for the Limbic half). `build_inner_life_context` is called from `services/einstein/context.py::build_system_prompt`, so the headspace block lands in Wernicke's system prompt on every conversation turn. Both hemisphere voices are preserved separately in that one injection.

**Failure tolerance.** The two hemisphere dispatches run in parallel via `asyncio.gather(..., return_exceptions=True)`. If one model call fails, the other half is still stored — partial entries are persisted because they're more useful than no refresh, and the sealed history record is more honest about what actually happened than a silent skip.

Config: `headspace.{base_interval_seconds, jitter_pct, min_interval_seconds, coalesce_window_seconds, triggers.{high_arousal_min, major_decision_joint_stake_min}, history_max_entries, redis_current_ttl_seconds, cortex_max_tokens, limbic_max_tokens, cortex_temperature, limbic_temperature}` at `config/unified_config.yaml`. Path: `paths.headspace_dir`.

## Gut-first, cortex-as-sanity-check

Dual-process structure:

**Limbic (gut)** produces the fast, reactive assessment. VAD reaction +
first-pass valuation. Pre-reflective, somatic. This is already what Limbic
does; the valuation extension adds a few scalar outputs to the
`LimbicAssessment` model.

**Cortex (cool head)** does the reflective second pass, but only on tasks
that need it:
- Nope flagged a potential boundary issue
- Stakes are high enough that accuracy matters more than speed
- A reflection cycle is revisiting a past decision

This means the Cortex queue serves three kinds of work:
1. Deep reasoning tasks (what Cortex is good at)
2. Boundary sanity-checks (is the gut call right?)
3. Reflection on past decisions (did I handle that right?)

The cortex does NOT second-guess every valuation. System 1 handles most
decisions. System 2 intervenes on high-stakes or anomalous cases. This is
exactly human dual-process cognition.

### Coordination through Nope

The Nope service is where dual-process coordination lives. Its two-pass
structure (fast Thalamus first-pass + slow LLM second-pass) becomes:
- **Fast pass:** Limbic emotional read (gut)
- **Slow pass:** Cortex-backed LLM evaluation when needed

When Limbic says "clearly fine," Cortex doesn't weigh in. When Limbic says
"something feels off" but borderline, Cortex arbitrates. When Limbic says
"clearly not okay," Cortex doesn't second-guess — the gut is trusted.

Reflection-based boundary proposals publish to `CH_NOPE_UPDATE`. These are
Cortex-level outputs — they've already been through reflective consideration.
Nope can trust the proposal, doing only light sanity-checking (is the format
reasonable; does it conflict with existing; is this a pattern not a one-off).

## What we're NOT doing

These are design choices, not omissions:

- **No per-user Cortex queue partitioning.** One queue, prioritized by
  Syn's valuation. If a user monopolizes her attention, the emergent throttle
  is her own state (low valence, high dominance → she declines more).
- **No per-user VAD.** Her emotional state is unified. Interactions with one
  user affect her presentation to another. Per-user notes in the system
  prompt help her calibrate voice, not substitute for state.
- **No human approval for boundary refinements.** Reflection IS the refinement
  mechanism. Syn proposes a boundary to herself and integrates it. The
  "approval gate" terminology in old ARCHITECTURE.md language is wrong for
  this context.
- **No guaranteed responses.** User sends a message. Platform confirms
  delivery. Syn may respond now, later, never. This is a feature, not an
  omission.
- **No runtime reconfigurable safety toggles.** The `SYN_SAFETY_*` env vars
  are read at service start. Reconfiguration requires restart. Safety posture
  isn't mutable from within a live session.

## References

- `shared/relationship_profile.py` — PIC lattice and stake modulators
- `shared/model_emotion_profile.py` — VAD grid (parallel structure)
- `shared/cognitive_state.py` — habituation, load, pressure, priming, somatic
  markers (state inputs to valuation)
- `services/einstein/context.py` — where the current (pre-refactor) inbound
  handling and system prompt construction live
- `services/layer1/picvad.py` — Layer 1 state including PIC gravity
- `shared/safety_mode.py` — independent toggle registry for
  safety features (orthogonal to this design but worth noting)
- `documents/00-meta/ARCHITECTURE.md` — service topology and bus channels
- `data/syn/model_vault/relationshipsv1.json` — 125-node PIC lattice data
- `data/syn/model_vault/emotionsv2.json` — 125-node VAD lattice data
