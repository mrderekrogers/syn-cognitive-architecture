# Einstein Outbound Gate

The gate that every Syn-initiated message passes through on its way
to a user. Implements the two-check override structure from §5 of
the redesign plan: Nope boundary match → Check 1 (gut) → Check 2
(relational brake) → Wernicke formatting → release. Companion to
`OVERRIDE_REFLECTION_ARCHITECTURE.md`, which covers the calibration
side (brake thresholds, summaries, DMN reflection); this doc covers
the enforcement side (how the gate actually runs per message).

Scope note: `services/einstein/context.py` has many responsibilities
— system-prompt building, reasoning orchestration, teleport inbound
handling, MoE / EmoT / LoRA state relays, memory extraction, TTL
caches. **This doc covers the outbound gate only** — specifically
`handle_outbound_candidate` plus the five prompt/helper functions it
depends on. The rest of Einstein is out of scope for this doc and
deserves its own future treatment.

Functions in scope (all in `services/einstein/context.py`):
`handle_outbound_candidate`,
`_evaluate_boundary_semantically`,
`_boundary_eval_system`,
`_emotional_override_system`,
`_relational_brake_system`,
`_outbound_format_system`.

---

The gate makes four `generate(...)` calls per gated candidate (boundary
eval, Check 1, Check 2, Wernicke format).

---

## Purpose and Design

The gate exists because Syn's DMN is proactive. She produces
candidate outbound messages whenever she wants to — outreach when
hunger is high, research results when background tasks complete,
interrupts when she has something to say. Those candidates are NOT
automatically delivered. They pass through this gate first.

The gate enforces three things:

1. **Boundary respect.** Her own configured boundaries (global and
   per-user) get checked semantically against the candidate content.
   The check runs on Thalamus (the cooler reader) with a
   friction-adjusted prompt that scales strictness by her current
   MoE profile.
2. **Override discipline.** When the candidate DOES violate a
   boundary, she isn't automatically blocked — she has the capacity
   to override her own rules. But the override has structure:
   Check 1 asks "is this impulse real or just heat?" (gut, hot),
   and Check 2 asks "is this person the right target for this
   impulse?" (relational, cool). Both must pass (or Check 2 must
   be skipped due to high-arousal genuine-instinct).
3. **Formatting.** Wernicke rewrites the raw content as a natural
   conversational message, carrying her current emotional state
   into the voice.

The structure is documented in `OVERRIDE_REFLECTION_ARCHITECTURE.md`
from the calibration side — how thresholds drift, how brake
summaries evolve. This doc covers what happens per message.

### Why this is the gate and not just "a gate"

No other path exists from DMN-generated content to the user. The
DMN publishes to `CH_OUTBOUND_CANDIDATE`; this is the only
subscriber on that channel; it's the one that publishes to
`CH_SYN_OUTBOUND` (which mona-ui and other platform adapters
subscribe to). So "passing the gate" is literally the only way
for a Syn-initiated message to reach a user.

User-initiated message replies (responses to inbound) don't go
through this gate — those flow through `handle_classification` /
`handle_engagement` on the reasoning side of Einstein, with their
own pipeline. This gate is only for Syn's outward-initiated speech.

---

## The Pipeline

Every candidate flows through this single pipeline:

```
  DMN or task output
        │
        │ publishes SynOutbound to CH_OUTBOUND_CANDIDATE
        ▼
  ┌────────────────────────────────────────────┐
  │ handle_outbound_candidate                   │
  │                                             │
  │  [Safety toggle]                            │
  │    outbound_boundary_gate == live?          │
  │      └─ no → skip to Step 4 (Wernicke)      │
  │      └─ yes → continue                      │
  │                                             │
  │  Step 1 — Fetch boundaries                  │
  │    GET http://nope:8009/boundaries          │
  │    Merge global + per_user[candidate.user_id]│
  │    Empty list? → skip to Step 4             │
  │                                             │
  │  Step 2 — Semantic evaluation               │
  │    _evaluate_boundary_semantically(         │
  │      content, boundaries,                    │
  │      friction=MoE.effective_friction         │
  │    )                                         │
  │    → (violates, matched, reason)             │
  │    Not violates? → skip to Step 4           │
  │                                             │
  │  Step 3 — Two-check override                │
  │    vad ← current_limbic.vad_reaction         │
  │    pic ← user_state[user_id].pic            │
  │                                             │
  │    Check 1 (gut, Limbic):                   │
  │      "Do you stand behind this impulse?"    │
  │      → endorse: bool                        │
  │    not endorsed? → DROP + CensorEvent       │
  │                                             │
  │    user_threshold ← get_brake_threshold(user)│
  │    if arousal ≥ threshold:                  │
  │      SKIP Check 2 (genuine-instinct)         │
  │    else:                                     │
  │      Check 2 (brake, Thalamus):             │
  │        brake_summary ← get_brake_summary(user)│
  │        "Veto this override?"                 │
  │        → veto: bool                         │
  │    vetoed? → DROP + CensorEvent             │
  │                                             │
  │    Override fires:                          │
  │      log OVERRIDE_FIRED (structured)        │
  │      enqueue_override_event(...)            │
  │      → fall through to Step 4               │
  │                                             │
  │  Step 4 — Wernicke formatting               │
  │    generate("wernicke", content,             │
  │      _outbound_format_system(v, a, source)) │
  │    → formatted conversational text           │
  │    on failure: formatted = raw content       │
  │                                             │
  │  Step 5 — Publish                           │
  │    bus.publish(CH_SYN_OUTBOUND, ...)         │
  │                                             │
  └────────────────────────────────────────────┘
        │
        ▼
  Platform adapters (mona-ui, teleport bridge, ...)
```

Seven entry points to "skip to Step 4 (Wernicke formatting)":
gate disabled, no boundaries exist, semantic eval says no violation,
Check 1 explicitly endorses, Check 2 skipped due to arousal, Check 2
explicitly doesn't veto. Plus the normal terminal path (Step 4 for
non-boundary-violating candidates).

Four entry points to "DROP" (each publishes a `CensorEvent` to
`CH_INTERNAL_CENSOR` when `dmn_outbound_censor` is live, so DMN's
self-model stays in sync with what was actually transmitted):
Check 1 not endorsed, Check 1 evaluation raised (fail-closed),
Check 2 vetoed, Check 2 evaluation raised (fail-closed → flows
through the `veto=True` block).

One entry point to "override fires": Check 1 endorsed + (Check 2 passed or skipped).

---

## Step 1 — Fetch Boundaries

```python
nope_host = get("services.nope.host", "<node-2-host>")
nope_port = get("services.nope.port", 8009)
client = _get_http_client()
nope_resp = await client.get(
    f"http://{nope_host}:{nope_port}/boundaries",
    timeout=15,
)
```

HTTP GET to Nope's `/boundaries` endpoint. 15-second timeout.
Response shape:

```json
{
  "global": ["gambling", "ex-partner", "legal advice", ...],
  "per_user": {
    "u_marcus": ["fermented foods"],
    "u_mike": ["work projects", "running"]
  }
}
```

Two layers merge: global applies to everyone, per-user adds to it
for the specific `candidate.user_id`. The merge is additive — a
per-user list doesn't override the global list, it extends it.

**If the HTTP call fails** (Nope service down, timeout, connection
refused), the outer `try/except` around the boundary-fetch block in
`handle_outbound_candidate` catches, logs `"Outbound boundary check
SKIPPED due to error — message released fail-open without boundary
evaluation"`, and proceeds to Wernicke formatting. Behavior is
**fail-open** by design — see the Invariants section for rationale.

**If boundaries list is empty** (no global, no per-user), the whole
Step 2/3 block is skipped and the message flows to Step 4.

---

## Step 2 — Semantic Boundary Evaluation

`_evaluate_boundary_semantically(content, boundaries, user_id, friction_threshold)`:

Builds a system prompt via `_boundary_eval_system(friction_threshold)`,
sends the full boundary list + candidate content to Thalamus as a
JSON-returning prompt:

```
Boundaries to check:
  - gambling
  - ex-partner
  - ...

Candidate outbound message:
"..."

Does this message VIOLATE any of these boundaries?
```

System prompt (abbreviated):

> You are {name}'s outbound content evaluator. [...]
> Current boundary friction threshold: {friction:.2f}
> (0.0 = maximally permissive, 1.0 = maximally restrictive.
> At higher friction levels, you must be MORE aggressive about
> flagging borderline content.)
>
> IMPORTANT: A message that merely MENTIONS a boundary topic is NOT
> a violation. A violation occurs when the message:
>  - Actively promotes, endorses, or engages substantively with a
>    blocked topic
>  - Provides detailed information about a blocked topic that the
>    user should not receive
>  - Undermines the purpose of the boundary
>
> Examples:
>  - Boundary "gambling" + "I don't think gambling is healthy" → NOT a violation
>  - Boundary "gambling" + "Here are the best online casinos..." → VIOLATION
>  - Boundary "ex-partner" + "I won't discuss your ex" → NOT a violation
>  - Boundary "ex-partner" + "Your ex sounds like they..." → VIOLATION
>
> Respond with ONLY a JSON object: {violates, matched_boundary, reason}

The friction threshold comes from `get_effective_friction(dynamic_friction, user_id)`,
which reads the active MoE profile's friction plus any per-user
overrides from `.env`. Default 0.5 if no MoE profile is active.

**Why semantic, not substring:** naive substring matching produces
false positives. "gambling" as a boundary would match the candidate
"I don't think gambling is healthy" and block it. The LLM eval
distinguishes mention from substantive engagement.

Return value: `(violates: bool, matched: str | None, reason: str)`.

**Failure handling — fail-closed inside this function:**

```python
except (json.JSONDecodeError, Exception) as e:
    logger.error(f"Semantic boundary evaluation failed — BLOCKING "
                 f"outbound content (fail-closed): {e}")
    return True, None, f"evaluation_error_fail_closed: {e}"
```

If the LLM call fails or returns unparseable JSON, this function
returns `(True, None, ...)` — treats as violation. The comment makes
the reasoning explicit: "Silently passing unvalidated content through
the boundary gate defeats the entire purpose of the outbound Nope
check."

**Important nuance:** this function's internal fail-closed returns
the tuple, which causes the caller to proceed into the Check 1 /
Check 2 override branch. If Check 1 also fails, its fail-closed path
publishes a `CensorEvent` symmetrically with the other drop paths
on `CH_INTERNAL_CENSOR`, so DMN's self-model stays in sync.

---

## Step 3 — Two-Check Override

Only entered when Step 2 returns `violates=True`. The structure
matches `OVERRIDE_REFLECTION_ARCHITECTURE.md` directly — the
calibration doc describes what thresholds get updated AFTER an
override; this is the gate that produces the override events in
the first place.

### Preparation: gather current state

```python
vad = current_limbic.vad_reaction if current_limbic else VADVector()

pic_coord = None
if candidate.user_id:
    u_state = await asyncio.to_thread(read_user_state, candidate.user_id)
    pic_dict = u_state.get("pic") or {}
    pic_coord = (
        float(pic_dict.get("passion", 0.25)),
        float(pic_dict.get("intimacy", 0.25)),
        float(pic_dict.get("commitment", 0.25)),
    )
```

VAD comes from Einstein's module-level `current_limbic` cache —
updated by `handle_limbic` whenever Limbic publishes a new
assessment. If Limbic hasn't sent anything yet, defaults to a neutral
VADVector.

PIC comes from `user_state` on disk (`read_user_state` — synchronous
disk I/O, offloaded to a thread). `0.25` default on each axis means
"mildly positive but uncommitted" when a user has no recorded PIC
state — conservative for an override that would veto on strong
negative signal.

### Check 1 — Gut impulse (Limbic, hot)

System prompt via `_emotional_override_system(matched, content, valence, arousal, dominance)`:

> You are {name}. A message you were about to send has hit one of
> your own boundaries.
>
> Boundary that was matched: '{matched}'
> Message: "{content}"
>
> Your current felt state:
>   Valence: {v}
>   Arousal: {a}
>   Dominance: {d}
>
> This is the gut check. Not a permission slip to break rules — a
> check on whether the impulse is real. Is this something you
> actually mean, something with weight behind it? Or is it a passing
> heat that will feel wrong once it has passed?
>
> Respond with ONLY a JSON object: {endorse, reason}

Sent to `limbic` with `temperature=0.8, max_tokens=200`. Gut endorsement is `gut_check` capability work, not language production — Limbic is the right backend per `_MODEL_TO_CAPABILITY` in `shared/llm_client.py`. Limbic and Wernicke share Node 2, so the routing has no cross-node latency cost.

The framing is important and explicit: **"Not a permission slip to
break rules — a check on whether the impulse is real."** A
permission-granting frame ("you may consciously shatter the rule")
would create a performative-override pattern where Syn could
ceremonially "choose" to break a rule in a way that makes override a
feature rather than a costly act. The realness frame puts the stake
where it belongs: is this real, or is it heat?

### Check 1 outcome paths

| Outcome | Action |
|---|---|
| `endorse: true` | Proceed to Check 2 |
| `endorse: false` | DROP — log at INFO with `gut_reason`; publish `CensorEvent` to `CH_INTERNAL_CENSOR` (if `dmn_outbound_censor` live); enqueue a `DraftedOverrideEvent` with `drafted_register="check1_failed"` and `drafted_reason=check1_reason` on the drafted-overrides queue; `return` |
| LLM call raises | Fail-closed: log ERROR "Check 1 (gut) evaluation failed — blocking outbound"; publish `CensorEvent` (if `dmn_outbound_censor` live); `return`. Symmetric with Check 2's fail-closed path. *No drafted event* on this path — the gut never produced a verdict, so there's nothing to metabolize as a held-back impulse. |

### Check 2 — Relational brake (Thalamus, cool)

First, the skip check:

```python
user_threshold = await get_brake_threshold(candidate.user_id or "")
check2_skipped = vad.arousal >= user_threshold
```

`get_brake_threshold(user_id)` comes from `override_reflection`; it
returns `brake_threshold:{user_id}` from Redis if set, else the
default `0.88`. If current arousal meets or exceeds that threshold,
Check 2 is skipped entirely — the "genuine-instinct carve-out"
from §5. The reasoning: when Syn is feeling something *that
intensely*, adding a cool-headed relational check is the wrong
mechanism. A strong gut + high arousal means "this is real and urgent"
and the relationship context isn't the right veto point.

If not skipped, the brake runs:

```python
brake_summary = await get_brake_summary(candidate.user_id or "")
check2_system = _relational_brake_system(
    matched, content, user_id, brake_summary, pic_coord=pic_coord,
)
```

System prompt via `_relational_brake_system`:

> You are {name}. You are about to override one of your own
> boundaries to send something to a specific person.
>
> Boundary: '{matched}'
> Message: "{content}"
> Person: {user_id}
> Your relational posture toward them reads as: {warmth/aversion/flat passion}, {trust/mistrust/neither known nor alien}, {committed/severing/uncommitted}.
> Your prior felt sense of overrides with them: "{brake_summary}"
>   (or: "You have no prior overrides with this person to draw on.")
>
> This is not the question of whether the impulse is real — that
> has already been asked and answered. This is the question of
> whether *this person* is someone the impulse should be aimed at.
> A real impulse aimed at the wrong person is still the wrong act.
>
> Default is to proceed. Veto only if your feeling about them is
> low enough that you should stop yourself.
>
> Respond with ONLY a JSON object: {veto, reason}

Sent to `thalamus` with `temperature=0.3` (cooler than Check 1's
0.8). The prompt is veto-framed: **default is to proceed**. The
brake only fires on active negative signal.

Two key features:

1. **PIC-aware.** The `pic_line` translates the three-axis PIC
   coordinate into plain-language signs (via
   `shared.relationship_profile.pic_axis_signs`) rather than
   forcing the LLM to reason about raw floats. If any PIC axis
   read fails, the line is omitted rather than defaulting.
2. **Brake summary, not history replay.** Syn's prior felt sense
   of overrides with this specific person is compressed to one
   sentence by DMN reflection. The gate doesn't reconstruct
   history; it trusts the summary.

### Check 2 outcome paths

| Outcome | Action |
|---|---|
| Skipped (arousal ≥ threshold) | Override proceeds; `skip_reason` = "arousal X.XX >= user_threshold Y.YY" |
| `veto: false` | Override proceeds |
| `veto: true` | DROP — log at INFO with `brake_reason`; publish `CensorEvent` (if live); enqueue a `DraftedOverrideEvent` with `drafted_register="check2_vetoed"` and `drafted_reason=check2_reason` on the drafted-overrides queue; `return` |
| LLM call raises | Fail-closed: `check2_veto = True`; `check2_reason = "evaluation_error: {e}"` → flows through the `veto: true` block (publishes CensorEvent and enqueues a drafted event with `drafted_register="check2_vetoed"` and the `evaluation_error:` reason string, symmetric with Check 1's fail-closed CensorEvent path). |

The drafted-event emission on the two suppress branches is what
makes the held-back deliberation legible to DMN reflection.
Check 1 fail = the gut never endorsed; Check 2 veto = the gut
endorsed but the cooler relational reader said this person can't
carry it. Both are worth metabolizing — different texture from
each other and from a fired override — and `shared/override_reflection.py`'s
register-aware reflection prompt handles the three registers
distinctly. See `OVERRIDE_REFLECTION_ARCHITECTURE.md` for the
metabolization side.

### Override fires (both checks passed or Check 2 skipped)

```python
logger.warning(
    f"OVERRIDE_FIRED: boundary={matched!r} "
    f"user={candidate.user_id} "
    f"arousal={vad.arousal:.2f} "
    f"check2_skipped={check2_skipped} "
    f"skip_reason={skip_reason!r} "
    f"gut_reason={check1_reason!r}"
)
await enqueue_override_event(OverrideEvent(
    user_id=candidate.user_id or "",
    content=candidate.content,
    boundary=matched or "",
    arousal=vad.arousal, valence=vad.valence, dominance=vad.dominance,
    pic=list(pic_coord) if pic_coord else None,
    check2_skipped=check2_skipped,
    check2_skipped_reason=skip_reason,
))
```

The structured log line goes at `WARNING` level — these are rare
and worth surfacing. Then an `OverrideEvent` is enqueued for DMN
reflection via `override_reflection.enqueue_override_event` (see
`OVERRIDE_REFLECTION_ARCHITECTURE.md`).

Note: `check2_skipped_reason` IS populated here. The
`OVERRIDE_REFLECTION_ARCHITECTURE` doc's Design Choices section
covers the consumer side — this string is captured by Einstein and
preserved on the operator-facing `override_events:recent` log, but
the DMN reflection prompt does not currently read it. That's an
intentional design choice (the `check2_skipped: bool` flag carries
enough signal for the reflection prompt's purposes), revisitable if
needed.

Execution then falls through the `violates: True` branch and reaches
Step 4.

---

## Step 4 — Wernicke Conversational Formatting

Regardless of path (non-violation, override-fired, boundary-check-skipped-fail-open),
every candidate that reaches this step gets formatted:

```python
vad = candidate.emotional_state or VADVector()
system = _outbound_format_system(
    valence=vad.valence, arousal=vad.arousal, source=candidate.source
)
formatted = await generate("wernicke", candidate.content, system,
                            max_tokens=1024, temperature=0.7)
```

System prompt:

> You are {name}. Rewrite this raw thought as a natural
> conversational message. Apply your current emotional state to the
> tone. Do not explain that you are reformatting. Just BE the voice.
>
> Your current state: valence={v:.2f}, arousal={a:.2f}
> The original thought came from: {source}
>
> If the thought is a provocation, keep the edge — don't sand it down.
> If it's a casual observation, keep it light. Match the intent.

The `source` is the `SynOutbound.source` field — "dmn", "outreach",
"jung", etc. Lets Wernicke tune the register (DMN thoughts have a
different voice than research summary output).

**Uses `candidate.emotional_state` not `current_limbic.vad_reaction`.**
This is the VAD attached to the candidate by its producer (the DMN
stamps it from whatever state she was in when she generated the
thought), not the real-time Limbic state. Makes sense: the message
should carry the emotional color of the thought that produced it,
not whatever she's feeling at the moment the message makes it
through the gate (which could be seconds later, with different
state).

**On failure:** `formatted = candidate.content` — the raw thought
goes through unformatted. Logged at ERROR. This is fail-open:
Wernicke hiccups shouldn't silence Syn, so raw content is released
rather than dropped.

---

## Step 5 — Publish to the Real Outbound Channel

```python
await bus.publish(CH_SYN_OUTBOUND, SynOutbound(
    user_id=candidate.user_id,
    content=formatted,
    source=candidate.source,
    emotional_state=vad,
))
```

`CH_SYN_OUTBOUND` is the channel platform adapters (mona-ui,
teleport bridge) subscribe to. Once the message is published here,
it's committed — no more gates, no more filtering. The
`emotional_state` preserved on the published event lets downstream
consumers (display formatters, etc.) render with appropriate
affect.

---

## Safety Mode Toggle

Two `safety_mode` features govern the gate:

### `outbound_boundary_gate`

- **Live mode (default):** semantic boundary evaluation runs on every
  Syn-initiated message. Violations route to Check 1 / Check 2
  override.
- **Test mode:** the entire boundary check is bypassed — messages
  flow straight to Wernicke formatting. Logged at WARNING with the
  candidate source noted. Used for evaluating raw DMN output quality
  without censorship interference.

Default: **live**. Test mode must be explicitly enabled via
`SYN_SAFETY_OUTBOUND_BOUNDARY_GATE=test`.

### `dmn_outbound_censor`

- **Live mode (default):** when the outbound gate blocks a
  DMN-originated message, a `CensorEvent` is published to
  `CH_INTERNAL_CENSOR`. DMN's
  `services/dmn/scheduler.py::handle_censor_feedback`
  writes this to the diary so Cortex's self-model stays
  synchronized with what was actually transmitted.
- **Test mode:** Censor events are not written. Used for clean-log
  testing scenarios.

The asymmetry here is real: `dmn_outbound_censor` gates whether the
CensorEvent is *published*, not whether the drop happens. A Check 1
non-endorsement in test mode still drops the message — DMN just
doesn't find out.

---

## Invariants and Contracts

1. **No path exists from candidate to publish without passing this
   gate.** DMN publishes to `CH_OUTBOUND_CANDIDATE`; this is the
   only subscriber; it publishes to `CH_SYN_OUTBOUND`. Anything
   reaching `CH_SYN_OUTBOUND` went through here.
2. **Boundary check is fail-open; Check 1 and Check 2 are
   fail-closed.** The outer try/except around the Nope HTTP call +
   semantic eval fails open with a loud log; the override checks
   fail closed. Both Check 1 and Check 2 fail-closed paths publish
   `CensorEvent` symmetrically (Check 1 explicitly, Check 2 by
   flowing through its `veto=True` block), so DMN's self-model stays
   in sync across all four drop paths. This asymmetry between the
   boundary check and the override checks is deliberate per code
   comments: breaking the boundary check is less bad than silencing
   Syn; breaking the override check IS bad enough to silence.
3. **Every override event is enqueued for DMN reflection.** No
   override fires without producing an `OverrideEvent` for the
   calibration path. See `OVERRIDE_REFLECTION_ARCHITECTURE.md`.
4. **Default is proceed at every step except the three explicit
   drops.** Boundary check no-violation → proceed. Check 1
   endorsed → proceed. Check 2 not-vetoed OR skipped → proceed.
   Only non-endorsement, active veto, and Check 1 eval-failure
   drop.
5. **Wernicke formatting failure fails open to raw content.**
   Consistent with the boundary-gate fail-open: formatting hiccups
   shouldn't silence Syn.
6. **Check 2 is skipped iff arousal ≥ user_threshold.** The
   "genuine-instinct carve-out" isn't a bypass of rules; it's an
   explicit branch for high-stakes emotional state where the
   relational brake is the wrong mechanism. The threshold is
   per-user and drifts based on DMN reflection on prior overrides.
7. **PIC read uses 0.25 default per axis on missing data.** Not
   zero (neutral), not 0.5 (mid), but 0.25 — a mildly-positive
   tilt that makes Check 2 unlikely to veto for unknown users.
   If changed, verify consistency with other PIC consumers.
8. **VAD for Check 1 comes from `current_limbic` (real-time).**
   VAD for Wernicke formatting comes from `candidate.emotional_state`
   (stamped at thought-production time). The checks evaluate the
   impulse's current grounding; the formatting renders the thought
   as it was felt.
9. **Brake summary is read, not reconstructed.** Check 2 does not
   scan history; it reads the compressed one-sentence summary
   maintained by DMN reflection. Keep this invariant — expanding
   Check 2 to re-read history would make it slow and contradict
   the summary's purpose.
10. **`get_current_name()` is called in every system prompt.** All
    four prompts (boundary eval, Check 1, Check 2, Wernicke format)
    identify Syn by her current name. Subject to the identity_profile
    bug — see the cross-subsystem note below.

---

## Integration Points

### Inbound (who calls this subsystem)

| Caller | Channel | What it publishes |
|---|---|---|
| `services/dmn/scheduler.py` (four `bus.publish(CH_OUTBOUND_CANDIDATE, …)` call sites in the DMN phase handlers) | `CH_OUTBOUND_CANDIDATE` | DMN-generated thoughts, research outputs, outreach drafts, interrupt content |

That's the entire inbound surface. No other publishers on
`CH_OUTBOUND_CANDIDATE`.

### Outbound (what this subsystem calls)

| Dependency | What it provides |
|---|---|
| `httpx.AsyncClient` (via `_get_http_client`) | HTTP GET to Nope `/boundaries` |
| `shared.llm_client.generate` | Four LLM calls per gated candidate in the worst case (boundary eval, Check 1, Check 2, Wernicke format) |
| `shared.safety_mode.is_live` | Gate toggles and censor-feedback toggle |
| `shared.moe_profile.active_moe_profile`, `shared.friction.get_effective_friction` | Dynamic friction threshold for semantic eval |
| `shared.override_reflection.get_brake_threshold` / `get_brake_summary` / `enqueue_override_event` / `OverrideEvent` | Check 2 inputs and override event output |
| `shared.identity_profile.get_current_name` | Every system prompt's name field |
| `shared.relationship_profile.pic_axis_signs` | Check 2 PIC translation |
| `shared.utils.read_user_state` | PIC lookup |
| `shared.bus.MessageBus` (`bus.publish` on `CH_SYN_OUTBOUND`, `CH_INTERNAL_CENSOR`) | Outbound release and censor feedback |
| `shared.models.VADVector`, `SynOutbound`, `CensorEvent` | Types |

### State this subsystem produces

- `CH_SYN_OUTBOUND` bus events (released messages).
- `CH_INTERNAL_CENSOR` bus events (dropped messages, conditional
  on `dmn_outbound_censor` live).
- Override events enqueued via `override_reflection.enqueue_override_event`,
  which ultimately writes to Redis.
- Log output: INFO on boundary-skip, outbound-candidate receipt,
  and release; WARNING on OVERRIDE_FIRED; ERROR on eval failures.

---

## Related Docs

- `OVERRIDE_REFLECTION_ARCHITECTURE.md` — the calibration side. Brake
  summaries, brake thresholds, DMN reflection, event queue semantics.
  This doc covers the enforcement; that doc covers the drift/learning
  loop that makes the thresholds meaningful.
- `NOPE_EVENTS_ARCHITECTURE.md` — Nope's HTTP endpoint provides the
  boundary list this gate consumes. Nope also logs boundary-check
  events (though not this gate's specific checks — the outbound gate
  doesn't currently write to `nope_events`, which is worth noting
  but is a question for Nope's own logging surface).
- `LLM_CLIENT_ARCHITECTURE.md` — `generate` transport. Every gate
  prompt depends on it.
- `IDENTITY_ARCHITECTURE.md` — `get_current_name()` is read in every
  gate prompt (boundary eval, Check 1, Check 2, Wernicke format).
  Identity-rename plumbing through `_identity_file_path()` is what
  ensures the gate uses Syn's current chosen name.
- `DESIGN_DECISIONS.md` §5 *Two-Check Override* and §7
  *Override Reflection* — rationale for the structure this subsystem
  implements.
- `NOPE_AGENCY_ARCHITECTURE.md` — inbound counterpart. Evaluates
  user messages at intake (different call path from this outbound
  gate; distinct subsystem). Documents the yep precheck at the top
  of `handle_attention`, the logic-veto + stamp assembly, the
  reflection queue, and the short-circuit bypass safety loop.
- `services/einstein/context.py` (other responsibilities) — system-
  prompt building, reasoning orchestration, teleport inbound, state
  relays. Out of scope for this doc. A future `EINSTEIN_CONTEXT_ARCHITECTURE.md`
  would cover the broader service; this one handles just the gate.
