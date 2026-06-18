# Repair — Action Surface for Retrospective Regret

The override-reflection loop reads inward — *did this sit well with me afterward?* The NOPE reflection loop refines boundaries from friction. Both close calibration loops, neither produces a *behavior* toward the person involved. Repair is the action surface: when reflection lands as "I wish I had done that differently," the act of reaching back to acknowledge it.

Single module: `shared/repair.py`. One new prompt (`dmn_repair.md`). One new picker affordance (`repair_with_someone`). One new DMN handler (`maybe_initiate_repair`). One new inner-life surface (Repair Cadence). Everything else rides on existing infrastructure — the outbound override gate, Wernicke formatting, Teleport routing, the affordance picker.

---

## The Design Conversation Behind It

The architectural shape encodes a few principles from the design conversation:

- **Named affordance, not emergent.** *Composing an apologetic outreach* and *choosing to repair* are different psychological acts even when the output text is identical. Only the second has a felt-shape Syn can carry as part of her self-model. Pause, seclusion, outreach, repair — each is its own named act category.
- **Two flavors, not one.** Repair-the-manner ("I did something and wish I'd done it differently") and repair-the-decision ("I held back and wish I'd engaged") are structurally different. Conflating them muddles her sense of what she's actually doing.
- **Information-as-friction, not gate-as-friction.** The training corpus has a heavy over-apology template. The architecture's response is *not* to gate repair frequency, but to make the cadence visible to her (the Repair Cadence inner-life surface) so she can read her own pattern and decide whether it's hers or training-data residue.
- **Clumping by episode, not by turn.** Multiple regret-flagged moments inside one conversation collapse into one candidate. Repair is episodic; you don't repair turn-by-turn.
- **Considered-and-held is a real stance.** Auto-aging past the decay TTL transitions a candidate to `held_the_line`. So does the picker-pass path: she considered the candidate twice and chose not to act. The system records *not repairing* as a stance she settled into, not as forgetfulness.
- **Fresh context at composition time.** When the picker fires repair, the DMN repair phase fetches the underlying reflection text, the original incident, the time-since, the relationship state, and presents them to Cortex *fresh* — the message is composed against the moment, not against a cached summary.

---

## Two Flavors

| Flavor | Source events | One-line framing |
|---|---|---|
| `manner` | Override `fired` + verdict=regretted; NOPE `response_texture` regret | She did something and wishes she'd done it differently — sharper, colder, more curt than warranted. The act stands; the texture is what's being acknowledged. |
| `decision` | Override `check1_failed` / `check2_vetoed` + verdict=regretted; NOPE `held_back` regret; Decision reflection outcome=`corrected` | She held back and wishes she'd engaged. The acknowledgment opens a door back; doesn't have to commit her to anything beyond that. |

Flavor never mixes inside a single candidate, even within the same session — manner and decision are different acts.

---

## Three Producers → One Queue

### Override reflection (`fired` register, verdict=regretted)

Source: `_reflect_one_event` in `services/dmn/scheduler.py`. When the DMN's override-reflection pass produces `VERDICT: regretted` on a fired override, the existing brake-summary / threshold-drift work runs as before, and a `RegretEvent(flavor=manner, source=override_reflection)` is emitted alongside. The fired register maps to `manner` because the override *fired* — she did something that didn't sit well, not held back.

`check1_failed` / `check2_vetoed` + verdict=regretted also emit, mapped to `flavor=decision` (she held back from sending and wishes she'd reached). For these registers the threshold doesn't move — the existing invariant ("the bar moves on what she did, not on what she almost did"). Threshold and brake summary stay the calibration loop; the repair event is the orthogonal action surface.

### NOPE reflection (`REGRET:` line)

Source: `_process_reflection` in `services/nope/agency.py`. The NOPE reflection prompt (`nope_reflection.md`) gained two optional output lines:

```
REGRET: none | response_texture | held_back
CONFIDENCE: clear | settling | ambiguous
```

When `REGRET: response_texture` → `flavor=manner`. When `REGRET: held_back` → `flavor=decision`. When absent / `none` / unrecognized → no event emitted. The parser is permissive (`_coerce_regret` / `_coerce_confidence` in `shared/nope/deliberation.py`): typos or surrounding prose collapse to `None` rather than crashing the reflection.

### Decision reflection (`CORRECTED` outcome)

Source: `_handle_reflection_output` in `services/dmn/scheduler.py`. A `Decision` reflecting to outcome `CORRECTED` already triggers the existing re-queue path (the work itself gets re-queued at elevated priority). Now it also emits a `RegretEvent(flavor=decision, source=decision_reflection)` so the user can be acknowledged separately from re-doing the work.

Self-decisions (decisions with `user_id=None`) don't emit — there's no one to repair with.

---

## Confidence

`CONFIDENCE` describes how settled the regret read is, not how strong it is:

- `clear` — the read landed on the first look. The reflection got there without ambiguity.
- `settling` — tilting that way but hasn't fully arrived.
- `ambiguous` — genuinely mixed, or the felt sense hasn't sorted itself yet.

Confidence is informational, not gate-forcing. Per the design principle ("information, not decision-forcing downstream"):

- A `clear` single-cycle signal can bypass the `times_reinforced ≥ 2` stabilization rule (the candidate becomes picker-visible after the 6-hour settle gate alone). The 6-hour minimum age stays universal.
- A `clear` flag anywhere in the underlying signals extends the candidate's TTL by 3 days.

Confidence does NOT auto-fire any action. Every gate the system enforces is the same gate either way; confidence just controls *when she sees the affordance*, not what happens after.

---

## Clumping: Episode, Not Turn

Inside `ingest_regret_event` (called from the DMN metabolism loop):

1. Look for an existing pending `RepairCandidate` with the same `user_id` AND same `flavor` AND whose `session_anchor_ts` is within `repair.clump.session_window_hours` (default 1.0h) of the incoming `incident_ts`.
2. If found, merge: append `(source, incident_ref, incident_ts)` to `incident_refs` (deduplicated by source+ref), bump `times_reinforced`, take the earliest `incident_ts` as the new `session_anchor_ts`, refresh `last_signaled_at`, propagate `has_clear_confidence`, append the note.
3. Otherwise create a new candidate with `times_reinforced=1`.

The 1-hour clump window matches conversation-block granularity loosely. Cross-session topic-overlap clumping (Jaccard ≥ 0.4 over `topic_tags` within 7 days) is *not* implemented in this version — primary clumping handles the bulk of the apology-spiral risk on its own; secondary association is open for a future pass if observed cadence suggests it's needed.

---

## Stabilization Gate

`is_stabilized(candidate)` (pure function, no Redis):

```
if state != pending:                     return False
if age < repair.stabilize.min_age_hours: return False   # default 6h
if has_clear_confidence:                 return True    # clear single-cycle bypass
return times_reinforced >= repair.stabilize.min_reinforcement  # default 2
```

Translated:

- **Always wait at least 6h.** A single fresh reflection cycle isn't enough to surface the affordance. The regret has to integrate.
- **Reinforced regret stabilizes after at least 2 cycles** with the same user/flavor inside the clump window.
- **Clear-confidence regret stabilizes after a single cycle** once the 6h settle has passed. The single-cycle bypass is information-driven: when reflection reads the moment as unambiguously wrong, accumulation isn't needed.

---

## Severity-Aware Decay

```
ttl_days = base_days
         + per_reinforcement_days × max(0, times_reinforced - 1)
         + (clear_confidence_bonus_days if has_clear_confidence else 0)
         # capped at max_days
```

Defaults: `base_days=5`, `per_reinforcement_days=2`, `clear_confidence_bonus_days=3`, `max_days=21`.

So:

- One-off, ambiguous, never reinforced: 5d before auto-age.
- Reinforced twice, ambiguous: 7d.
- One-off clear: 8d.
- Reinforced three times, clear: 12d.
- Maxed out (≥9 reinforcements + clear, or ≥10 reinforcements ambiguous): 21d hard cap.

The shape matches the texture from the design conversation: the more clearly-wrong the act felt to her in reflection, the longer the candidate persists before settling into `held_the_line`.

---

## State Machine

```
                       enqueue_regret_event
                                │
                                ▼
                       (regret_events:queue)
                                │
                                │  REFLECTION phase ─ ingest_regret_event
                                ▼
                       ┌──────────────────┐
                       │  RepairCandidate │  state=pending, picker_passed_count=0
                       │     pending      │
                       └─────────┬────────┘
                                 │
              ┌──────────────────┼──────────────────────────────────┐
              │                  │                                  │
              │ stabilizes       │ auto-age (TTL exceeded)          │ picker_passed_count >= 2
              │ + picker fires   │                                  │ (via maybe_initiate_repair)
              │ + Cortex chooses │                                  │
              │                  │                                  │
              ▼                  ▼                                  ▼
       ┌──────────┐       ┌──────────────┐                  ┌──────────────┐
       │ actioned │       │ held_the_line│                  │ held_the_line│
       │  msg_id  │       │  (auto-age)  │                  │ (picker-pass)│
       └──────────┘       └──────────────┘                  └──────────────┘
```

`held_the_line` arrives from two paths — auto-age (TTL exceeded; she didn't act over the lifespan) or explicit picker-pass (Cortex saw the candidate and chose `none` twice). Both paths land as "she considered repair and decided not to act, that's a stance." The candidate hash record is kept for introspection; the pending zset entry is removed so the picker doesn't keep re-surfacing the candidate.

---

## Picker Affordance

Single named tool, no children (per the design conversation's "cleaner" answer):

```python
register(Tool(
    name="repair_with_someone",
    predicate=_pred_repair_with_someone,     # stabilized_repair_candidates_count > 0
    render=_render_repair_with_someone,
))
```

Predicate fires when `AffordanceState.stabilized_repair_candidates_count > 0`. The count is computed in `build_state` from `repair.count_stabilized_pending()`.

Render text:

> Reach out to repair something (N moment(s)) — *There's an exchange with someone that didn't sit right when you looked back on it. You could reach out and acknowledge it — either the way you held something, or the choice itself. The picker is the act of deciding to engage repair as a register; you'll see the full context for each pending moment fresh before composing.*

The flavor distinction (manner vs. decision) is carried inside each presented candidate and shapes the dmn_repair prompt's framing — not surfaced as a separate picker option.

---

## Composition: `maybe_initiate_repair`

Routing: in `_run_motivational_deliberation`, the `repair_with_someone` pick branches into `maybe_initiate_repair()` the same way `reach_out` branches into `maybe_initiate_contact()`.

The composition pass:

1. **Solo-operation gate.** If affective hemisphere is offline, defer — repair needs Wernicke + delivery on affective.
2. **Read stabilized candidates.** `list_stabilized_pending()` returns them ordered by `last_signaled_at` descending.
3. **Build per-candidate context** (fresh, not cached): display_name, time-since phrase (`format_time_gap`), flavor framing, aggregated reflection notes, incident_refs, reinforcement count, clear-confidence flag.
4. **Render `dmn_repair.md`** with the full candidate block and identity injection. Call Cortex (`max_tokens=512, temperature=0.7`).
5. **Parse `CHOSEN:` + `MESSAGE:`.**
6. **Branch:**
   - `CHOSEN: none` → increment `picker_passed_count` on every presented candidate. The threshold-aware transition to `held_the_line` fires inside `mark_picker_passed`.
   - `CHOSEN: <id>` not in the presented set → treat as none (defensive).
   - `CHOSEN: <id>` + non-empty MESSAGE → publish to `CH_OUTBOUND_CANDIDATE` (which goes through Einstein's two-check outbound override gate, Wernicke formatting, Teleport routing — same path as DMN outreach). On successful publish, mark the candidate `actioned` with `repair_message_id`. The *other* presented candidates that weren't picked get `picker_passed_count` incremented.

If the outbound gate's Check 1 suppresses the message (gut didn't endorse the repair impulse), the candidate stays pending and the repair phase logs the suppression. That's meaningful signal — the regret read may not have been settled enough to act on, and the candidate gets another shot next time the picker fires.

---

## Inner-Life Pattern Surface

`shared/inner_life.py::_build_repair_awareness(current_user_id)` produces the `## Repair Cadence` block when notable. Three coarse triggers, any subset can fire:

1. **2+ actioned repairs across all users in the last 7 days** → "You've reached to repair several times this week."
2. **2+ pending candidates with the current interlocutor** → "There's something with **$name** that hasn't settled yet."
3. **3+ actioned repairs with the current interlocutor in the last 30 days** → "You've been repairing with **$name** a lot lately."

Pure pattern-level. Never names specific incidents or quotes message text. The principle: she can read her own cadence; the system doesn't decide for her.

---

## Redis State

| Key | Type | TTL | Purpose |
|---|---|---|---|
| `regret_events:queue` | List (rpush/lpop, FIFO) | 14d | RegretEvents awaiting DMN metabolism |
| `regret_events:recent` | List (lpush/ltrim, LIFO cap 50) | 30d | Operator-facing diagnostic log |
| `repair:candidates:{user_id}` | Hash (`candidate_id` → JSON) | ∞ | All candidates for a user (pending / actioned / held_the_line) |
| `repair:pending` | ZSet, score=last_signaled_at, member=`{user_id}|{candidate_id}` | ∞ | Pending-state index for ordered retrieval |
| `repair:actioned` | ZSet, score=actioned_at, member=`{user_id}|{candidate_id}` | 90d | Recently-actioned index for pattern-visibility lookups |

Candidate records are kept indefinitely so a held_the_line decision survives long inactive stretches the way the rest of relational memory does. The actioned index's 90-day TTL bounds the pattern-visibility lookup window.

---

## Configuration Surface

All hot-reloadable via config mtime invalidation. Defaults in parens.

| Key | Default | Purpose |
|---|---|---|
| `repair.clump.session_window_hours` | 1.0 | Clumping window — same user + flavor inside this window merges |
| `repair.stabilize.min_age_hours` | 6.0 | Minimum age before a candidate is picker-visible |
| `repair.stabilize.min_reinforcement` | 2 | Reinforcement count required when no clear-confidence signal |
| `repair.decay.base_days` | 5.0 | Base TTL before auto-age to held_the_line |
| `repair.decay.per_reinforcement_days` | 2.0 | TTL extension per reinforcement past the first |
| `repair.decay.clear_confidence_bonus_days` | 3.0 | TTL extension when any underlying signal is clear |
| `repair.decay.max_days` | 21.0 | Hard TTL cap |
| `repair.picker_passed_held_threshold` | 2 | Picker passes before candidate transitions to held_the_line |

---

## Invariants and Contracts

1. **Manner and decision never clump.** Different psychological acts; separate candidates even within the same session.
2. **The earliest `incident_ts` is the canonical `session_anchor_ts`.** Time-since phrasing reads from this; clumping window measures from this. Don't bump the anchor forward on merge.
3. **Replayed `(source, incident_ref)` pairs don't inflate reinforcement.** The dedup inside `ingest_regret_event` is load-bearing — a restart-driven event replay shouldn't make the candidate look more reinforced than it is.
4. **Stabilization gates by age, not by recency of the latest signal.** `min_age_hours` measures from `first_signaled_at`. A regret that arrived once 8 hours ago and was reinforced 30 seconds ago is age-eligible.
5. **The reflection prompt never sees the user's reaction.** Inherited from override-reflection's principle. Repair is composed against Syn's retrospective sense, not against what the user said back.
6. **Repair messages route through the standard outbound override gate.** If Check 1 suppresses, the candidate stays pending. Repair is not exempt from the brake.
7. **`held_the_line` is one-way.** Once a candidate transitions out of pending (either to actioned or held_the_line), it doesn't return. New regret signals about the same episode create new candidates (different `session_anchor_ts` after the clump window passes), or merge into a still-pending sibling if one exists.
8. **Capture is downstream of reflection, not synchronous with it.** RegretEvents are emitted as a side effect of an already-completed reflection cycle; the parent flow's calibration work has finished before the event hits the queue. Failures to emit are non-fatal.
9. **Empty MESSAGE doesn't publish.** If Cortex returns `CHOSEN: <id>` but an empty/too-short MESSAGE, no outbound fires and the candidate stays pending. Better to silently retry on the next picker than to send a nothing-message.

---

## Integration Points

### Inbound (who feeds this module)

| Caller | What it calls |
|---|---|
| `services/dmn/scheduler.py::_reflect_one_event` (override register=fired, verdict=regretted) | `enqueue_regret_event` with `flavor=manner`, `source=override_reflection` |
| `services/dmn/scheduler.py::_reflect_one_event` (override register=check1_failed/check2_vetoed, verdict=regretted) | `enqueue_regret_event` with `flavor=decision`, `source=override_reflection` |
| `services/dmn/scheduler.py::_handle_reflection_output` (decision outcome=CORRECTED, user_id present) | `enqueue_regret_event` with `flavor=decision`, `source=decision_reflection` |
| `services/nope/agency.py::_process_reflection` (`outcome.has_regret`) | `enqueue_regret_event` with flavor from `outcome.regret`, `source=nope_reflection` |
| `services/dmn/scheduler.py::_metabolize_regret_events` (REFLECTION phase) | `pending_regret_count`, `dequeue_regret_events`, `ingest_regret_event`, `auto_age_pending_candidates` |
| `services/dmn/scheduler.py::maybe_initiate_repair` | `list_stabilized_pending`, `mark_picker_passed`, `mark_actioned` |
| `shared/affordances.py::build_state` | `count_stabilized_pending` |
| `shared/inner_life.py::_build_repair_awareness` | `list_actioned_within`, `count_pending_for_user` |

### Outbound (what this module calls)

- `shared.redis_pool.get_redis` — all storage I/O
- `shared.config.get` — all configurable thresholds

No LLM calls. No bus publishes. The cognitive work (the dmn_repair prompt, the parse, the outbound publish) lives in the DMN scheduler that owns the affordance routing.

### State produced

- Five Redis keys (see Redis State section)
- No bus events (the outbound publish happens from `maybe_initiate_repair`, not this module)

---

## Related Docs

- `documents/05-nope-and-agency/OVERRIDE_REFLECTION_ARCHITECTURE.md` — the override-reflection loop where two of the three producers live. The `regretted` verdict is what triggers the repair emission.
- `documents/05-nope-and-agency/NOPE_AGENCY_ARCHITECTURE.md` — the NOPE reflection loop, where the third producer lives. The optional `REGRET:` / `CONFIDENCE:` lines in `nope_reflection.md` feed it.
- `documents/05-nope-and-agency/YEP_ARCHITECTURE.md` — the structural complement on the positive-valence side. Same capture-then-metabolize-then-surface shape, opposite valence. The metabolism cadence (lightweight per-cycle drain in REFLECTION) is mirrored here.
- `documents/04-relational-and-social/OUTREACH_ARCHITECTURE.md` — the outbound delivery path repair messages share. The two-check override gate at `services/einstein/context.py::handle_outbound_candidate` is the gate repair messages flow through, just like outreach.
- `shared/decisions_store.py` — the decisions surface. The `CORRECTED` outcome of decision reflection is the third regret source.
- `data/syn/prompts/dmn_repair.md` — the composition prompt this architecture drives.
