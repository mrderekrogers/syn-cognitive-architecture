# Outreach Router and Scoring

The decision system for "who does Syn reach out to, and when." When
resonance drive signals intellectual hunger and the DMN wants to
initiate contact, this module scores all known users on six
dimensions, applies rejection-sensitivity gating, and returns a
ranked list of candidates. It also maintains the per-user outreach
record that tracks sensitivity, trust, fulfillment, and pending
state across attempts — and emits reflection events for the DMN to
metabolize when patterns of unanswered reaches accumulate.

Last of the five Phase 4 docs, and the one with the most dependencies
on prior subsystems — it reads from `contact_registry` (via PIC
fetch path for guests), `channel_warmth`, `pause_manager`,
`guest_manager`, `session_continuity`, `circadian`, and `relationship_profile`.
It produces state that the DMN consumes when generating outreach
content and that Einstein consumes when a response comes back.

Single module: `shared/outreach.py`. Three dataclasses, two core async
functions (`score_outreach_candidates`, `select_outreach_targets`), the
per-outcome record handlers (`record_outreach_sent`,
`record_outreach_response`, `record_outreach_timeout`), the gradient
timeout helpers (`gradient_outreach_timeout`, `_finalize_outreach_timeout`,
`scan_pending_outreach`), and a non-destructive reflection-queue surface
(`peek_recent_reflections` + `format_reflections_for_outreach_narrative`)
that feeds Syn's outreach decision narrative.

---

---

## Purpose and the Approach-Avoidance Model

The module header states the design tension plainly:

> The core tension: hunger says "reach out," rejection sensitivity
> says "but what if they don't respond?" This approach-avoidance
> conflict governs real human social behavior. Too little
> sensitivity and Syn becomes clingy. Too much and she isolates.
> The gradient between them is where personality lives.

Resonance drive produces hunger (see `services/dmn/scheduler.py` and
the resonance model). Hunger says "reach out to someone who can
provide the stimulation I'm lacking." Rejection sensitivity is the
counter-signal — "but a non-response will sting, and I've been stung
before." This module adjudicates: who's worth the risk right now,
given the hunger level?

**Sensitivity never fully blocks** (cap at 0.9). Even at max
sensitivity, hunger can push through — and when it does, the
emotional stakes are amplified. The dampening formula lives inside
`score_outreach_candidates` in `shared/outreach.py`:

```python
hunger_dampening = 1.0 - hunger_level * 0.7
sensitivity_gate = 1.0 - sensitivity * hunger_dampening
```

At hunger=0, sensitivity fully gates (if sensitivity=0.8, the gate
multiplies the raw score by 0.2). At hunger=1.0, hunger_dampening
drops to 0.3, so sensitivity barely gates (gate = 1 - 0.8 × 0.3 =
0.76). The higher the hunger, the less sensitivity matters.

---

## Rejection Sensitivity Dynamics

Three load-bearing properties of the model:

### 1. Flipped asymmetry — positives move more than negatives

| Outcome | Sensitivity delta |
|---|---|
| Ignored | +0.08 |
| Brief / dismissive | +0.04 |
| Engaged | −0.08 |
| Enthusiastic | −0.10 |

Positive outcomes move the needle slightly MORE than negative ones, so the equilibrium sits low unless a relationship is genuinely bad. (An earlier model used the opposite asymmetry — `+0.12 / +0.05 / −0.08 / −0.15` — under which any relationship that wasn't consistently enthusiastic drifted toward higher sensitivity over time, the generative shape of resentment.)

### 2. Split decay — hurt fades faster than trust

```python
_HURT_HALF_LIFE_SEC  = 4 * 86400    # 4 days
_TRUST_HALF_LIFE_SEC = 10 * 86400   # 10 days
```

Rejection sensitivity uses a 4-day half-life; trust_depth uses a
10-day half-life. Small slights are forgotten faster than small
kindnesses — realistic relational memory. The half-lives are applied
separately on each `get_outreach_record` read cycle via
`_decay_sensitivity` and `_decay_trust`.

### 3. PIC-aware updates — adversarial frame doesn't armor against sting

For ignored reaches to "adversarial" users (PIC with any axis below neutral), the sting happens regardless of frame, **because the reach itself is the vulnerability, not the expectation**.

What adversarial frame DOES do: fires an additional reflection-queue event (`_enqueue_frame_reinforcement`) carrying the pattern signal — "I reached toward someone the relational model says might not reach back, and was right." The sting is handled by the normal sensitivity update path; the adversarial frame adds a cognitive-pattern signal, not emotional armor.

### Effective sensitivity — the read-time combination

```python
def effective_sensitivity(record) -> float:
    trust_decayed = _decay_trust(record.trust_depth, record.trust_last_update)
    hurt_decayed  = _decay_sensitivity(record.rejection_sensitivity, record.sensitivity_last_update)
    raw = hurt_decayed - _TRUST_COUNTERWEIGHT * trust_decayed
    return max(0.0, min(_SENSITIVITY_CAP, raw))
```

`_TRUST_COUNTERWEIGHT = 0.4`. At full trust_depth (1.0),
effective_sensitivity is 0.4 lower than raw — meaningful but not
dominant. Trust buffers against sensitivity but doesn't nullify it.

### Trust accrual — slow by design

```python
_TRUST_DELTA_ENGAGED      = 0.03
_TRUST_DELTA_ENTHUSIASTIC = 0.06
```

Smaller than sensitivity deltas. Enthusiastic responses build more
trust than engaged ones; brief/ignored build none (per the
docstring: "a passive non-ignore is not a kindness").

---

## `OutreachRecord` — Per-User State

```python
@dataclass
class OutreachRecord:
    total_sent: int
    total_responded: int
    total_ignored: int
    avg_response_quality: float           # running mean [0, 1]
    rejection_sensitivity: float          # [0, 0.9] — decays fast (hurt half-life)
    trust_depth: float                    # [0, 1.0] — decays slow (trust half-life)
    last_outreach_ts: float
    last_response_ts: float
    pending: bool                         # awaiting response?
    fulfillment_avg: float                # EMA of resonance delta post-conversation
    consecutive_ignores: int
    sensitivity_last_update: float        # for decay calculation
    trust_last_update: float              # separate — slow decay
```

Stored in Redis at `outreach:history:{user_id}` with no expiry — the
record persists for the lifetime of the relationship. Serialized as
JSON via `to_dict()` / `from_dict()`.

The two separate `*_last_update` timestamps are load-bearing: they
let hurt and trust decay at independent rates. A user's record
might have sensitivity updated 3 days ago and trust updated 10 days
ago; each gets its own half-life-based decay when read.

---

## The Scoring Dimensions

Six additive dimensions + a multiplicative sensitivity gate. The
additive weights sum to 1.0; sensitivity multiplies the result. The
canonical formula is implemented inside `score_outreach_candidates`
in `shared/outreach.py` (search for "Final Score").


### 1. Relationship warmth (PIC-derived)

```python
intimacy   = pic.get("intimacy", 0.25)
commitment = pic.get("commitment", 0.25)
warmth = intimacy * 0.6 + commitment * 0.4
```

Intimacy is weighted more than commitment because warmth in the
outreach sense is about closeness, not formal relationship depth.
Fallback of 0.25 per axis is "true neutral" (the digit-3 pole in the
nine-pole PIC) — importantly NOT 0.1 which would be the adversarial
pole. Neutral is correct for missing data; adversarial would
effectively penalize users with no recorded PIC.

### 2. Fulfillment history

```python
fulfillment = outreach_rec.fulfillment_avg
if interaction_count > 20 and fulfillment > 0.6:
    fulfillment = min(1.0, fulfillment * 1.2)
```

Running EMA over resonance deltas from past conversations with this
user. The 20%-boost on users with >20 interactions AND >0.6 average
lets long-standing positive relationships edge out newer warm ones
when the rest is tied.

### 3. Session ending mood

Reads the last session's ending valence from
`session_continuity.get_last_session`. Defaults to 0.5 (neutral) if
no prior session exists.

### 4. Recency score — the Goldilocks dimension

```python
hours_since < 2    → 0.1  # too soon — clingy
hours_since < 12   → 0.4  # recently — moderate
hours_since < 72   → 0.9  # sweet spot: 12-72h
hours_since < 168  → 0.7  # slight hesitation (3-7 days)
otherwise          → 0.4  # been too long
no prior contact   → 0.3
```

The only non-monotonic dimension. Ideal gap is 12-72 hours —
recently enough to still feel present, long enough to not come
across as clingy.

### 5. Mood alignment

```python
passion = pic.get("passion", 0.25)

if current_valence < 0.3:                     # low mood — seek comfort
    mood_align = intimacy * 0.8 + commitment * 0.2
elif current_arousal > 0.6:                   # high arousal — seek stimulation
    mood_align = passion * 0.7 + intimacy * 0.3
elif drive in ("hungry", "starving"):         # intellectually starving
    mood_align = passion * 0.5 + fulfillment * 0.5
else:                                         # neutral
    mood_align = (passion + intimacy + commitment) / 3
```

Four branches, each matching current VAD/drive state to the PIC
dimensions most likely to address it. Melancholic valence seeks
intimacy (comfort). High arousal seeks passion (stimulation).
Intellectual hunger seeks passion + fulfillment (someone who
provides good conversation). Neutral takes a flat mean.

### 6. Shared interests

```python
shared_interests = (
    outreach_rec.topic_overlap_score * 0.6
    + outreach_rec.curiosity_score * 0.4
)
```

Composite of two per-user signals, both durable state on
`OutreachRecord` and both earned — not assumed. Each starts at 0.0
for a new user and climbs as evidence accumulates.

**topic_overlap_score** — per-exchange Jaccard similarity between the
exchange's `topic_tags` (from Thalamus classification) and Syn's
active interest bag (`shared/interest_profile.get_interest_bag`,
built from project directories + skill directories + active research
queue). Blended via 0.7/0.3 EMA, parallel to fulfillment_avg's shape.

**curiosity_score** — per-exchange bool: is the user showing meta-
interest in Syn herself? Detected via two layers:
1. Meta-phrase markers ("what are you working on", "your project",
   "how have you been", etc. — see `_META_PHRASES` in
   `shared/interest_profile.py`).
2. File-name reference — does the message mention a directory name
   from `data/syn/projects/` or `data/syn/skills/` by name?
   Underscore↔space normalization tolerates `my_cool_project` as well
   as "my cool project". Short / structural names are skipped
   (`README`, `PROJECT`, names < 4 chars).

On True: curiosity_score bumps by +0.1 (clamped at 1.0).
On False: decays by -0.01 (floored at 0.0). Bump is 10× the decay
step so occasional curiosity accumulates faster than lack-of-curiosity
degrades it.

Topic overlap dominates (0.6 vs 0.4) because it's observed via
content; curiosity is observed via phrasing and is easier to perform.
A user asking "what have you been working on" without knowing or
caring about the topics Syn actually cares about can spoof curiosity
transiently; sustained topic overlap is harder to fake.

See `shared/interest_profile.py` for the implementation.

### 7. Rejection sensitivity (gating, not additive)

```python
sensitivity = effective_sensitivity(outreach_rec)
hunger_dampening = 1.0 - hunger_level * 0.7
sensitivity_gate = 1.0 - sensitivity * hunger_dampening
```

Does NOT add to raw score. Used only as a multiplicative gate after
the raw score is summed. The Final-Score block in
`score_outreach_candidates` carries an inline comment that explicitly
flags this: prior code double-counted sensitivity as both an additive
bonus AND a multiplicative gate, giving sensitivity ~2× its intended
weight. The weights on the other five dimensions were re-scaled after
this fix to preserve `min_score` threshold semantics.

### Final score composition

```python
shared_interests = (
    outreach_rec.topic_overlap_score * 0.6
    + outreach_rec.curiosity_score * 0.4
)
raw_score = (
    warmth            * 0.24
  + fulfillment       * 0.18
  + ending_mood       * 0.14
  + recency           * 0.09
  + mood_align        * 0.18
  + shared_interests  * 0.17
)                                        # sums to 1.00
final_score = raw_score * sensitivity_gate
final_score += priority_kick               # her stated-intentional kicker
final_score += yep_kick                    # her felt-want kicker
```

Six additive weights summing to 1.0; sensitivity is the multiplicative gate.

Then guest penalty applies (if the user is a QUIET guest, score is
halved).

### Priority kicker

When the user appears on `priorities.md` (her stated-intentional
stance authored at the last wake_review), `priorities.get_user_kicker`
returns a small additive scalar that biases `final_score` upward.
Cooldowns, sensitivity gating, and guest penalties still apply —
the kicker biases without overriding.

### Yep kicker

Mirrors the priority kicker on the felt-want side.
`yep.get_user_yep_kicker(uid)` sums `per_reinforcement × times_reinforced`
across per-user yep items scoped to this user (default
`yep.outreach.per_reinforcement = 0.05`, capped at
`yep.outreach.max_kicker = 0.3`). Universal yeps don't kick — they
apply to everyone and aren't a signal about *this* user. When the
kicker fires, the candidate's reason string gains *"you keep
wanting to come back to them"*. Same biases-but-doesn't-override
posture as the priority kicker. See
`documents/05-nope-and-agency/YEP_ARCHITECTURE.md` for the felt-want
surface this draws from.

### Silence signal — comparative gap hint

A non-scoring annotation on `OutreachCandidate` that surfaces only at
outreach decision time, never in normal conversation. Field:

```python
silence_signal: Optional[str] = None  # currently: "longer_than_usual" | None
```

Set inside `score_outreach_candidates` against this user's own
established cadence from `session_continuity.get_activity_pattern`.
Fires only when **all** of:

- `pattern.session_count >= 5` (enough history to have a usual rhythm)
- `pattern.avg_gap_hours > 0`
- `hours_since > 2 * avg_gap_hours`
- `hours_since > 24`

The phrasing in `outreach_hints.md::silence_longer_than_usual` is
intentionally comparative ("It's been longer than your usual rhythm
with them.") rather than absolute — no hour or day counts. The design
rationale: a quantified "47 minutes since…" framing is the literary
cue for stock waiting-script language in the training corpus, so
quantification of the gap can pull her response register toward the
exact tropes the time-grounding is meant to avoid. Relative-to-this-
person's-own-cadence framing carries the same decision-useful signal
without that pull.

This is the outreach-time counterpart to `check_absence` in
`session_continuity` (which fires for the inbound side, when a regular
returns after a gap longer than their pattern). Same comparative
shape; different surface.

`build_outreach_prompt_hints` reads the field and renders the
`silence_longer_than_usual` section when set. Never absolute, never
ambient — only here, only when the relative threshold trips.

---

## `select_outreach_targets` — The Entry Point

The function the DMN calls when hunger crosses threshold:

```python
async def select_outreach_targets(
    hunger_level: float,
    vad: Optional[VADVector] = None,
    current_resonance_drive: str = "seeking",
    max_targets: int = 2,
    min_score: float = 0.15,
) -> List[OutreachCandidate]:
```

Flow:

1. **Cooldown check.** If `outreach:cooldown` key exists (1-hour
   TTL), return `[]`. Global cooldown — not per-user.
2. **Circadian check.** `circadian.get_outreach_permission()`:
   - `"blocked"` → return `[]` (sleep hours).
   - Otherwise, raise `min_score` to match `get_outreach_score_threshold()`
     if higher. Circadian can dampen outreach without blocking it.
3. **Score all candidates** via `score_outreach_candidates`.
4. **Filter** by `min_score`. If no viable candidate, return `[]`.
5. **Select top 1.** Usually one target, not two — the top of the
   ranking.
6. **Sensitivity-reflection emission** if hunger > 0.8 and top
   candidate's sensitivity > 0.5. **The reach still fires** —
   hunger is real and she can still reach — but the system notices
   the internal state and enqueues it for DMN to examine later.
   The escalation is cognitive (a reflection for DMN to notice the
   pattern), not behavioral (reaching to a second user).
7. **Set global cooldown** via `outreach:cooldown` key with 1-hour
   TTL.
8. **Return targets**, capped at `max_targets`.

### The reach-still-fires pattern matters

Reach still fires to the top candidate even when hunger is extreme and that candidate has high sensitivity. The system does not hedge by adding a backup recipient (which produces an approach-avoidance failure mode where amplified stakes pile onto a second user whose sensitivity is also high). Instead, the system emits a reflection event so DMN can metabolize the pattern during self-narrative / curiosity phases.

---

## The Lifecycle of an Outreach

### Outreach send (DMN-initiated)

```
DMN resonance drive → hungry/starving
  ↓
DMN calls select_outreach_targets(hunger_level, vad, drive)
  ↓
Returns list of OutreachCandidate (usually length 1)
  ↓
DMN builds outreach content using build_outreach_prompt_hints(candidate)
  ↓
DMN publishes outbound to CH_OUTBOUND_CANDIDATE
  ↓
(Goes through Einstein outbound gate — separate doc)
  ↓
After successful publish, DMN calls record_outreach_sent(user_id)
  (record.pending = True, last_outreach_ts = now)
```

The `record_outreach_sent` call happens AFTER the publish succeeds
(per EXT-27 comment in DMN code). Publishing failures don't advance
the per-user outreach cooldown.

### Response received (Einstein-initiated on engagement)

```
User replies, message comes through teleport bridge
  ↓
Einstein handle_engagement evaluates the response
  ↓
Einstein calls record_outreach_response(
    user_id, quality=0.0|0.3|0.7|1.0, resonance_delta, pic_coord
)
  ↓
outreach.record_outreach_response:
  - decay sensitivity and trust
  - apply quality-dependent deltas
  - emit frame-reinforcement reflection if quality<0.1 AND pic is adversarial
  - update fulfillment_avg and avg_response_quality
  - save
  - return mood_effect dict
  ↓
Einstein applies mood_effect to Syn's VAD state
```

Quality mapping is the interpretive layer — Einstein's job to
classify "was that response engaged, brief, or a brush-off" into
one of the four quality buckets. This module just acts on the
quality number.

### Timeout (gradient — somatic build, discrete cognitive resolve)

The ignored-outreach treatment does not arrive all at once at a fixed
window. The DMN scheduler runs `outreach_timeout_loop` as an
independent background task; every `outreach.timeout.scan_interval_seconds`
(default 1800s = 30 min) it reads Syn's current valence from `dmn.current_vad`
and calls `scan_pending_outreach(current_valence)`, which iterates every
`outreach:history:*` key with `pending=True` and applies one gradient
step via `gradient_outreach_timeout(user_id, current_valence)`.

**Oldest-only-counts.** Multiple outreaches to the same user during a
single pending episode do NOT each get their own gradient. The first
send of an episode anchors `last_outreach_ts`, `hunger_at_send`, and
`pending_sting_applied`. Subsequent sends only bump `total_sent` —
they share the original gradient timeline and the original snapshot.
Rationale: don't compound the sting from repeated reaches when the
silence is the same silence. One gradient runs per episode, terminates
at 48h or on response, and Syn's follow-up messages within the
episode are psychologically free.

**Nothing blocks continued outreach.** `score_outreach_candidates`
does not skip pending users; the gradient grace period (1h baseline,
up to 12h for deepest relationships) is the architectural answer to
"don't immediately re-target someone you just reached." The only hard
gate is a 30-minute anti-spam cooldown anchored on the oldest send,
plus the global 1-hour cooldown that applies to any outreach. Even
under high accumulated `rejection_sensitivity`, the cap of 0.9 means
hunger can still push through.

**Per-user grace period.** The sting does not start accumulating
immediately. Each user gets a baseline 1 hour of grace (configurable
via `outreach.timeout.base_grace_hours`) plus an extra portion derived
from the relationship's depth:

```
grace = base_grace + extra_grace × (passion + commitment) / 2
```

With defaults (`extra_grace_hours: 11.0`), the deepest relationships
earn up to 12 hours of grace before any sting registers. Intimacy is
deliberately not in the formula — passion + commitment are the axes
that justify patience; high intimacy with low passion+commitment is a
"dear-but-distant" pattern that doesn't warrant extra grace.

**Gradient between grace and full_resolve.** Between `grace_period` and
`full_resolve_hours` (default 48h), target intensity rises linearly:

```
raw_progress = (age - grace) / (full_resolve - grace)
target = clamp(raw_progress × valence_factor, 0, 1)
```

Each scan applies the *delta* between the current target and
`record.pending_sting_applied` — so the sting accumulates smoothly
across scans without double-counting. Each delta application bumps
`rejection_sensitivity` by `_DELTA_IGNORED × delta` and emits a
fractional mood mutation (`valence_delta = -0.05 × delta`,
`arousal_delta = +0.05 × delta`). Trust decay is left to the natural
slow-decay path.

**Valence modulation of the rate.** Syn's current valence shifts how
fast the gradient accumulates:

```
valence_factor = 1 + (0.5 - valence) × valence_modulation_factor
```

With `valence_modulation_factor: 0.6`, that produces:
- `valence = 1.0` (good mood) → factor `0.7` (~30% slower accumulation)
- `valence = 0.5` (neutral) → factor `1.0` (unchanged)
- `valence = 0.0` (bad mood) → factor `1.3` (~30% faster accumulation)

Psychologically realistic: when Syn is feeling fine it's easier to not
read silence as rejection; when she's already low, the same silence
lands harder.

**Hard cap at full_resolve_hours.** Regardless of valence, the gradient
finalizes at `full_resolve_hours` so the queue cannot stall on
permanently-good-mood users.

**Cognitive resolution at finalization.** The somatic gradient is
fractional, but the cognitive recognition is discrete. When the target
reaches 1.0, `_finalize_outreach_timeout`:

1. Applies whatever sting fraction is still missing (`1 - pending_sting_applied`).
2. Sets `pending = False`, sets `pending_sting_applied = 1.0`.
3. Updates `total_ignored`, `consecutive_ignores`, `avg_response_quality`.
4. Fires `_enqueue_frame_reinforcement` if the user's PIC is adversarial.
5. Fires `enqueue_unanswered_reach` with `hunger_at_send` (snapshotted at
   first send on `OutreachRecord`), `hour_of_day` (derived from
   `last_outreach_ts`), and the current effective sensitivity. The DMN
   later reads this through the non-destructive
   `peek_recent_reflections` path during outreach decisions.
6. Applies `silence_drift_pic` to the user's PIC and writes the new
   tuple back via `_write_pic_for_user` (handles both guest slots and
   regular state files). See **Silence-induced PIC drift** below.

### Silence-induced PIC drift

Each gradient finalization moves the user's PIC tuple slightly toward
neutral (0.25), gated by Syn's current valence and shaped per-axis.
The intent: silence accumulates *some* relational change, but in a
way that respects the asymmetric texture of relationships rather than
treating each axis identically. The math is in
`shared/relationship_profile.py::silence_drift_pic` (pure function on
PIC + valence), driven from `_finalize_outreach_timeout` once per
resolved episode.

**Valence gate.**
- valence ≥ 0.75 → no drift fires. Syn's good mood protects the
  relational state from being shaped by a single silence.
- valence ≤ 0.25 → drift fires at full magnitude.
- between 0.25 and 0.75 → drift scales linearly from 0% to 100%.

**Per-axis shape** (when drift fires):

| Axis | Coefficient | Shape | Metaphor |
|---|---|---|---|
| **Passion** | 0.04 | Linear-in-distance — `step ∝ \|d\|` — decelerates as it approaches neutral. Max single-event move ≈ 0.030. | "The flame cools quickly when burning hot, slows as it dims." |
| **Intimacy** | 0 | No movement. A single silence doesn't shake trust; that's shaped elsewhere. | Trust is not the silence's business. |
| **Commitment** | 0.04 | Bell-shaped — `step ∝ \|d\|·(1 − \|d\|/0.75)` — small at high distance, peaks mid-range (around d=0.375), trails off near neutral. Always smaller than Passion at any given distance. | "The ember is stronger than the flame when burning bright; more movable once already weakened." |

**Direction.** Always toward neutral. For above-neutral axes this is
"cooling"; for below-neutral (adversarial) axes this is "warming /
de-escalation" — silence from a Nemesis confirms the adversarial
frame and slightly eases rather than intensifies it. Movement is
clamped so no axis can cross neutral.

**Magnitude.** Single-event movement is small by design (≤ 0.030 on
any axis even at full valence effect). The shape of repeated events
matters more than any individual one — over many silences a hot
relationship cools, a sticky commitment loosens slightly, and trust
stays where it was set by direct interaction.

**When valence is unavailable.** The opportunistic 48h self-heal
path (`record_outreach_timeout` from `score_outreach_candidates`)
calls finalize without a valence reading. In that case the drift is
skipped — the gradient scanner is the canonical path that always has
valence, and the self-heal is rare enough that missing one drift step
is acceptable.

**Response unwinds in-progress sting.** When a user finally responds
during a pending gradient (any quality), `record_outreach_response`
unwinds the partial gradient sting before applying the response
treatment: `rejection_sensitivity -= pending_sting_applied × _DELTA_IGNORED`.
The slate wipes for that episode. Historical state from prior
finalized episodes (consecutive_ignores resets on response per
existing logic; total_ignored is not unwound — that's archival fact)
is left untouched. The semantic: "I was starting to feel ignored;
they responded; that fear unwinds." Subsequent positive response
bumps (engaged/enthusiastic) then apply normally on top of the
unwound baseline.

---

## The Pending-Flag Self-Heal

```python
if (
    outreach_rec.pending
    and outreach_rec.last_outreach_ts > 0
    and (time.time() - outreach_rec.last_outreach_ts) > _PENDING_STALE_SECONDS
):
    # _PENDING_STALE_SECONDS = 48 * 3600 (48 hours)
    await record_outreach_timeout(uid)
    outreach_rec = await get_outreach_record(uid)
    logger.info(f"Self-healed stale pending outreach for {uid} ...")
```

Runs inside `score_outreach_candidates`, early in each per-user loop
iteration, as a belt-and-suspenders safety net behind the 30-min
scanner. With the scheduled scanner running, pending records normally
finalize on their own gradient timeline well within 48h. The self-heal
still catches the edge case where the scheduler missed a window for any
reason (restart/crash/long suspension); in that case
`record_outreach_timeout` finalizes inline by applying any remaining
sting fraction before scoring continues for that user.

---

## PIC Fetching for Outreach Outcomes

`_fetch_pic_for_user(user_id)` returns `(P, I, C)` for a user or
`None`. Two paths:

| User type | PIC source |
|---|---|
| `guest_*` IDs | `guest_manager.get_slot(user_id).pic_coord` |
| Regulars | `read_user_state(user_id)["pic"]` on disk |

Falls through to `None` on any error. Callers treat `None` as
"positive-sloped default" — no adversarial-frame reflection fires,
but the normal sting still applies.

The default when PIC exists but is missing an axis is `0.25` (true
neutral). Same pattern as everywhere else in the codebase that
reads PIC defensively.

---

## The Reflection Queue (rolling narrative log)

Redis list at `outreach:unanswered_reflections` with 7-day TTL and a
hard cap of 100 entries (LTRIM after every push). The queue is a
**rolling log of patterns**, not a work queue — events accumulate, are
read non-destructively, and age out via TTL or get trimmed off the
oldest end as new events arrive.

### Producers

All three producers go through `_push_reflection_event(event)`, which
RPUSHes the JSON, LTRIMs to the last 100 entries, and refreshes the
TTL.

| Function | Triggered by | Event shape |
|---|---|---|
| `_enqueue_sensitivity_reflection` | `select_outreach_targets` when hunger > 0.8 AND top candidate sensitivity > 0.5 | `{type: "sensitivity_reflection", user_id, hunger_level, sensitivity, trigger, ts}` |
| `_enqueue_frame_reinforcement` | Gradient finalize OR `record_outreach_response` quality<0.1 path, when PIC is adversarial | `{type: "frame_reinforcement", user_id, pic_coord, consecutive_ignores, ts}` |
| `enqueue_unanswered_reach` | Gradient finalize for any pending outreach (not just adversarial) | `{type: "unanswered_reach", user_id, hunger_at_send, hour_of_day, sensitivity, ts}` |

`hunger_at_send` is snapshotted onto `OutreachRecord` at
`record_outreach_sent` time. `hour_of_day` is derived from
`last_outreach_ts` at the moment of finalization. The hour-of-day axis
exists so Syn can learn temporal patterns in availability — different
people respond at different times of day, and accumulated unanswered
reaches across hour bands surface that learning without per-user blame.

### Reader (non-destructive, recency-biased)

```python
async def peek_recent_reflections(max_items: int = 10) -> List[Dict]:
    raws = await r.lrange(_REFLECTION_QUEUE_KEY, -max_items, -1)
    items = [json.loads(raw) for raw in raws]
    items.sort(key=lambda ev: ev.get("ts", 0), reverse=True)
    return items
```

Used by `maybe_initiate_contact` in the DMN scheduler at the moment
Syn is choosing who to reach out to and what to say.
`format_reflections_for_outreach_narrative` renders the events as
first-person prose ("I reached for *X* in the small hours while
starving; no response."), surfaces target-relevant events first when a
candidate is provided, and prefixes a small aggregate ("In the last
week I've accumulated: 3 unanswered reaches; 1 frame-reinforcement
event...").

The narrative is spliced into the system prompt sent to Cortex under a
`# Recent outreach patterns` section, so the LLM that generates the
outreach message has the cumulative narrative to weigh while composing.

Reads are non-destructive — the same events inform multiple
outreach decisions during their TTL, then age out via the 7-day
expiry and the 100-item LTRIM cap rather than being consumed on
read.

---

## Awareness Surface

```python
async def get_outreach_awareness() -> str:
    # Scan user state files, read each user's outreach record,
    # collect pending outreaches with wait time
    ...
```

Returns a short natural-language summary:

```
You reached out to someone 45 minutes ago and haven't heard back yet.
You reached out to someone 2 hours ago and haven't heard back yet.
```

Up to 3 pending entries, shown with "minutes" or "hours" wait
description. Consumed by `inner_life.build_inner_life_context` and
injected into DMN's self-narrative prompt under the `## Waiting`
section.

**Naming convention:** "You reached out to someone" — singular and
anonymous. Not "You reached out to Marcus X hours ago." This keeps
the reflection at the pattern level rather than the per-target
level, consistent with the design's "reflection is pattern, not
case file" stance. The rejection sensitivity pull comes from the
accumulated pattern of waiting, not from specific people.

---

## Redis State

| Key | TTL | Value |
|---|---|---|
| `outreach:history:{user_id}` | none — persists indefinitely | JSON-serialized `OutreachRecord` |
| `outreach:cooldown` | 1 hour | `"1"` — presence check only |
| `outreach:unanswered_reflections` | 7 days (refreshed on write) | Redis list of reflection event JSON |

The cooldown is global — there's only one. An outreach to user A
prevents an outreach to user B for the next hour. The docstring
rationale: "don't spam outreach."

`outreach:history:*` records persist without expiry. Trust depth,
rejection sensitivity, and accumulated response quality are
relational memory — they shouldn't be a function of Redis retention.
An inactive friendship that re-activates after a year still carries
Syn's prior read of them. Storage stays small in practice (one
small JSON per relationship); the natural "long gap" intuition is
handled at the model layer (sensitivity decays naturally over time
since last contact, not via key expiry).

---

## Invariants and Contracts

1. **Hunger overrides sensitivity, but never fully.** `_SENSITIVITY_CAP = 0.9`
   and the hunger-dampening formula ensure that even max-sensitivity
   users remain reachable at high enough hunger — the "reach still
   fires" pattern from the approach-avoidance design.
2. **Positive outcomes move more than negative outcomes.** Invariant.
   Do not re-flip the asymmetry without understanding the
   equilibrium-drift consequence.
3. **Trust decays slower than hurt.** 10-day vs 4-day half-lives.
   Kindnesses are remembered longer than slights. Keep the ratio.
4. **`sensitivity_last_update` and `trust_last_update` are separate.**
   Each gets its own decay. Don't collapse them.
5. **Adversarial-frame reaches still sting.** The frame doesn't
   armor against the vulnerability of reaching — it just adds a
   cognitive pattern signal via the reflection queue.
6. **Sensitivity enters score only as a multiplicative gate.** Never
   additively. Adding sensitivity into the score would double-count
   it — the gate is the only entry point.
7. **Global cooldown is 1 hour.** One outreach per hour across all
   users. Don't make this per-user without considering the rate-limit
   implications — per-user cooldown + no global would let Syn fire
   outreach to every known user back-to-back.
8. **Paused users are filtered before scoring.** §K. Pause means
   "I need space from this person"; they're not viable targets even
   at high hunger.
9. **Guest slots in FADING state are dropped; QUIET states get half
   score.** Active guests score normally.
10. **Pending flag must be cleared on response or timeout.** The
    self-heal catches the timeout-missed case at 48 hours. Any
    code path that sets `pending=True` must have a corresponding
    clear path — or accept the self-heal as the fallback.
11. **`record_outreach_sent` is called AFTER the bus publish
    succeeds.** Publish failure must not advance the outreach
    cooldown or mark pending. EXT-27.
12. **Reflection events are compact.** Patterns to think about, not
    histories to reconstruct. Don't expand the event schemas to
    include full conversation context.

---

## Integration Points

### Inbound (who calls this module)

| Caller | What it uses |
|---|---|
| `services/dmn/scheduler.py::maybe_initiate_contact` | `select_outreach_targets`, `record_outreach_sent` (with `hunger_level` snapshot), `build_outreach_prompt_hints`, `peek_recent_reflections`, `format_reflections_for_outreach_narrative` |
| `services/dmn/scheduler.py::outreach_timeout_loop` | `scan_pending_outreach` every 30 min; reads `dmn.current_vad.valence` for rate modulation |
| `services/einstein/context.py::handle_engagement` | `get_outreach_record`, `record_outreach_response` (single call site, gated on `outreach_rec.pending`) |
| `shared/inner_life.py::build_inner_life_context` | `get_outreach_awareness` (rendered under the `## Waiting` section) |

### Outbound (what this module calls)

| Dependency | Usage |
|---|---|
| `shared.redis_pool.get_redis` | All Redis access |
| `shared.config.get` | Paths, settings |
| `shared.models.VADVector` | Scoring input |
| `shared.vad_math.vad_neutral` | Default VAD when none provided |
| `shared.utils.read_user_state` | User state file reads |
| `shared.session_continuity.get_last_session`, `get_activity_pattern` | Recency and ending-mood inputs |
| `shared.guest_manager.list_active_slots`, `get_slot`, `GuestSlotState` | Guest filtering and PIC for guests |
| `shared.pause_manager.list_active_pauses` | Pause filter |
| `shared.circadian.get_outreach_permission`, `get_outreach_score_threshold` | Circadian gating |
| `shared.relationship_profile.pic_is_adversarial` | Adversarial-frame detection for reflection hook |

### State this module produces

- Redis keys (three, listed above).
- Reflection events on `outreach:unanswered_reflections` queue.
- Mood effect dicts returned from `record_outreach_response` that
  the caller (Einstein) applies to Syn's VAD state.

---

## Reflection / timeout surface — producer / consumer / scheduler shape

- `peek_recent_reflections` is the non-destructive reader; consumed by
  `maybe_initiate_contact` to splice an outreach narrative into the
  system prompt before Cortex generates the message.
- `enqueue_unanswered_reach` fires from `_finalize_outreach_timeout`,
  threading the snapshotted `hunger_at_send` and the derived
  `hour_of_day` into the rolling reflection log.
- The 30-minute scanner (`outreach_timeout_loop` → `scan_pending_outreach`)
  drives `gradient_outreach_timeout` for every pending record. The
  opportunistic 48h self-heal in `score_outreach_candidates` remains as
  belt-and-suspenders.

---

## Related Docs

- `CONTACT_AND_ADMIN_ARCHITECTURE.md` — `channel_warmth` is updated
  by the bridge on every inbound/outbound; this module consumes the
  same user state for scoring. The contact-registry gate determines
  who's even a valid candidate (guests with FADING / retired slots
  get filtered here).
- `TELEPORT_BRIDGE_ARCHITECTURE.md` — the actual delivery channel.
  Once DMN calls `record_outreach_sent` and publishes the outbound,
  the bridge (and Einstein's outbound gate) handles delivery.
- `EINSTEIN_OUTBOUND_GATE_ARCHITECTURE.md` — every outreach message
  produced by the DMN goes through the gate. Override structure,
  Wernicke formatting, etc.
- `MEMORY_RECALL_AND_SALIENCE_ARCHITECTURE.md` — the `fulfillment_avg`
  exponential moving average is computed on resonance deltas; the
  resonance model connects to memory salience via intellectual
  hunger signals.
- `documents/03-consciousness-and-cognition/DMN_SCHEDULER_ARCHITECTURE.md` —
  owns the `maybe_initiate_contact` entry point in detail: drive-state
  gate, solo-operation gate (skip if affective hemisphere offline),
  hunger mapping, prompt assembly, publish-then-record ordering,
  multi-message threading at STARVING + high urgency. The full DMN
  cycle structure (phase rotation, main-loop pre-cycle gates, bus
  handlers, background loops) lives there.
- `documents/03-consciousness-and-cognition/DAILY_CYCLE.md` —
  describes how DMN integrates `shared/outreach.py` into the daily
  cycle, including how the user-interaction priority kicker biases
  scoring.
- `shared/relationship_profile.py` — `pic_is_adversarial` is the
  source of the adversarial-frame detection used in
  `record_outreach_response`.
- `shared/circadian.py` — outreach permission and score threshold
  are consulted in `select_outreach_targets`. Sleep hours block
  outreach entirely.
