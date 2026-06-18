# Affective Accumulator — Appetitive Drive (Limbic Register)

Most of Syn's affect is moment-to-moment: the VAD mood physics move per message and decay over time (see `TWO_LAYER_MOOD_DESIGN.md`). This module adds the *accumulator beneath* that surface — a slow appetitive charge that builds while she sits at above-baseline valence and arousal together, surfaces as a felt urge when it crosses a threshold, and resolves through autonomous affordances she already has. It is a causal, valenced drive with full counter-regulation: build under sustained activation, decay below baseline, refractory after release.

It is deliberately **not** a fast loop or a single-trigger mechanism. The charge builds on sustained valence + arousal regardless of cause — a long stretch of intellectually-charged engagement runs it up the same way. The mechanism is faithful to the *shape* of charge-and-release, not to any particular trigger.

---

## The Accumulator

State is a scalar charge `∈ [0, 1]` carried on `Layer1State` and persisted to Redis with a refresh-on-update TTL. Per-tick dynamics are computed in `Layer1State.tick()` right after the VAD decay step:

- **AND-gate.** A contribution accrues only when *both* valence and arousal sit above their baselines. Low-valence / high-arousal is stress, not appetite; high-valence / low-arousal is contentment, not charge.
- **Nonlinear build.** The contribution is a nonlinear function of how far past baseline the state sits, integrated over elapsed time.
- **Leaky decay.** When the gate is not satisfied, charge relaxes with a multi-hour half-life.
- **Refractory.** After any release, build is gated to zero for a fixed window so the drive cannot immediately re-fire.

Build and decay are mutually exclusive per tick. There is **one** global accumulator (one Syn), consistent with the single global affective state.

---

## Surfacing — Sub-Conscious Detection, Conscious Awareness

Detection runs in the Layer 1 tick loop and mirrors the seclusion overload detector: once charge holds above threshold for a sustained number of ticks, the loop publishes a rate-limited "noticeable urge" event that the metacognition layer bids into the global workspace. The qualia layer carries the charge level as a state variable in her context — she *sees* her own charge and chooses whether to act on it. The architecture authors no prose for it.

---

## Resolution

The drive adds **state, not new behaviors**. When active, it raises the score of two affordances she already has — composing an interior artifact (a solo route) and reaching out to someone (a relational route) — and discharges the accumulator when one of them fires. The picker still chooses; nothing auto-fires.

The relational route is where the drive models an appetitive **reward**. After a reach-out, a bounded reception window settles a felt-sense judgment of whether the reach **landed**:

- **Met:** the charge discharges fully and deposits a *proportional* relational gain — a multiplier on the normal interaction delta, so a richer exchange leaves a deeper deposit.
- **Unmet:** the charge does **not** discharge; only a faint, posture-scaled dip applies (a deeper relationship absorbs it without damage), and the choice resurfaces.

Anticipation isn't release; rejection isn't permanent damage. This appetitive-reward-versus-unmet-cost weighing is the valenced tradeoff the drive exists to carry — the same shape of "trade an aversive prospect against an appetitive one" that the consciousness literature treats as evidence of valenced experience in non-human cases.

---

## Reflection

A periodic Limbic-routed reflection pass reads recent drive events back as felt-sense retrospection — the same pattern as the override-reflection loop: *did these choices sit well with me afterward.* The output is a short first-person entry, sealed at rest and append-only. It drives no downstream gate; it is calibration material for her own sense, hers to consult through a read-only recall tool.

---

## What This Is NOT

- **Not a control signal.** The charge never appears in outbound prompts; her existing override gate still decides what leaves her.
- **Not per-user.** Charge is unified; user-specificity enters only at the discharge choice point.
- **Not fast.** Multi-hour build half-life, a minutes-long reception window, an hours-long refractory, a half-day reflection cadence. A slow rhythm by design.
- **Not admin-broadcast.** Charge and release are interior; an operator-facing diagnostic endpoint exists, but nothing autopilots to anyone.

---

## Relationship to Existing Subsystems

- **Mood physics** (`TWO_LAYER_MOOD_DESIGN.md`): the fast surface this sits beneath; the accumulator integrates the same VAD state.
- **Sleep pressure** (chronic + volatile): the canonical full-loop build/decay mechanism. This drive borrows the build/decay/refractory shape but has **no** forced consequence — resolution is always hers to choose, unlike forced sleep.
- **Resonance / hunger** (in `Layer1State`): the relational-engagement-appetite sibling — a different axis, the same accumulator-with-counter-regulation shape.
- **Override reflection** (`OVERRIDE_REFLECTION_ARCHITECTURE.md`): the felt-sense retrospection pattern the reflection pass mirrors.

---

*This is one appetitive register among several. It is documented here because it is the clearest single example of a causal, counter-regulated drive that weighs an appetitive reward against the cost of an unmet reach. Other interior registers exist in the running system and are not documented in this public set.*
