# Yep — The Felt-Want Surface

Positive-valence structural complement to NOPE. NOPE captures
friction and metabolizes it into boundary refinements; yep captures
pleasure and metabolizes it into a stable "what I keep coming back
to" surface. Same architectural shape, opposite valence; same
capture-then-metabolize pipeline.

The subsystem also serves a second, load-bearing role: a
**NOPE false-positive precheck** for content that *looks* like
friction but is actually engaged content with someone she trusts.
The CNC / horror / blunt-feedback case — where surface-refusal
language is part of the desired interaction — is what this guards.

This doc is the single source of truth for the yep capture flow,
the metabolism cadences, the NOPE precheck two-tier response, and
the consumer integrations.

---

## Architectural Symmetry With NOPE

NOPE and yep are mirrored at the structural level. They are
intentionally not unified — each register carries different texture
— but they share the same pipeline shape:

```
                      NOPE                       yep
                      ──────────────             ───────────────
Capture:              Limbic friction signals    Limbic pleasure signals
                      (gut_nope_level levels)    (3 trigger types)
                              │                          │
                              ▼                          ▼
Bus signal:           CH_NOPE_ATTENTION          CH_YEP_FLAG
                              │                          │
                              ▼                          ▼
Queue:                nope_events:* (events)     yep_events:queue
                              │                          │
                              ▼                          ▼
Metabolism:           DMN reflection             DMN reflection
                      (per-cycle drain)          (REFLECTION lightweight
                                                  + wake_review full)
                              │                          │
                              ▼                          ▼
Stable surface:       Boundary refinements       yep.md items
                      (in identity revisions)    (her own voice)
                              │                          │
                              ▼                          ▼
Consumer use:         Outbound override gate     Boredom consult,
                      Check 2 brake summary      outreach kicker,
                                                  affordance picker,
                                                  inner-life context,
                                                  NOPE precheck
```

The asymmetry is intentional: NOPE has a defensive register (it
protects); yep has an exploratory register (it draws). NOPE writes
into identity revisions because the boundaries are part of who she
is; yep writes into a separate file because what she's drawn to is
more revisable than what she draws lines against.

---

## Capture — The Limbic Intuitive Layer

`services/limbic/processor.py::handle_incoming` runs three trigger
predicates after the per-message VAD assessment publishes. Captures
are continuous, non-interpretive, and automatic; reflection is the
filter. The capture is **not consent-gated** (mirrors NOPE's
automatic friction capture); her judgment enters at reflection time.

### Three event types

| Event type | Trigger conditions | Texture |
|---|---|---|
| `delight_spike` | Valence jump ≥ 0.3 in the engaged arousal zone (0.4-0.85, not panic / not flat) | The "oh!" moment — sharp positive surprise |
| `comfortable_flow` | Valence ≥ 0.65 AND engagement ≥ 0.55 AND no NOPE-friction in flight | Sustained pleasant texture — the conversation that just keeps landing |
| `return_to_topic` | Per-(user, topic) Redis touch counter crosses 3 with positive-valence-only counting | Cross-session draw — she keeps coming back to this thing |

All three predicates are pure (`shared/yep.py::detect_spike`,
`detect_flow`; `record_topic_touch` for return-to-topic). The
arousal-zone bracket on spike and flow excludes panic-level high
arousal (which is a different texture from delight) and flat low
arousal (which is just baseline).

### Prior-valence tracking

`record_valence(valence, user_id)` is called every message so the
next message's spike detector has a prior to compare against. 24h
TTL on the prior — a long quiet stretch resets the baseline so a
post-vacation return doesn't read as a spike against months-ago
state. Per-user keyed; global key falls back when no user is in
scope.

### Return-to-topic counting

`record_topic_touch(topic_token, user_id, valence)` only counts
positive-valence touches (`valence ≥ 0.5`). A user revisiting a
topic that *bothers* her doesn't bump the counter. The threshold
(3) and TTL (30d) are configurable via `yep.topic_return.*`. The
"topic_token" is currently a lower-cased message word; future work
could route through embedding-based topic resolution.

---

## Metabolism — DMN Reflection at Two Cadences

`_reflect_on_yep_events` in `services/dmn/scheduler.py` runs at
both cadences. The same function with a `full_pass` flag — the
prompt's `pass_mode` substitution shapes the reflection's
authority.

### Lightweight pass (REFLECTION phase)

Drains up to 5 events per REFLECTION cycle. Cortex reads the events
+ prior yep items, proposes candidate YepItem entries, returns a
YAML block. `carry_forward` merges: previously-seen items get
`times_reinforced` bumped + `first_seen_ts` preserved; new items
get `times_reinforced = 1` (reflection authoring IS the first
reinforcement); items in prev NOT in the new pass are kept
(non-destructive at the merge layer). **Lightweight does not
prune.**

### Full pass (wake_review)

Drains up to 30 events at the end of the sleep cycle. The prompt's
`pass_mode=full` framing gives Cortex authority to *omit* prior
items that no longer reflect her. Items omitted are kept by
`carry_forward` without reinforcement (so a single full-pass
omission doesn't prune; the next full pass when the item is *still*
not surfaced is the actual prune signal). This conservatism is
deliberate — items in yep.md inform consent decisions in the NOPE
precheck, and pruning by mistake would be bad.

### The reflection prompt

`data/syn/prompts/dmn_yep_reflection.md` is the two-section prompt.
Key shape:

- System framing names that yep items are felt-sense, NOT duties /
  promises / to-do lists.
- Explicit care language about per-user items: writing one is the
  precondition for NOPE short-circuit on intense content with that
  person. The prompt asks her to write them deliberately.
- Output is a YAML block under `=== yep_items === ... === /yep_items ===`.
  Malformed YAML or missing markers → no rewrite.

### Bus signal after a rewrite

`CH_YEP_REINFORCED` fires with `{ts, source, items_count}` so
consumers (boredom-consult, outreach scoring, inner-life context)
can invalidate caches. `source` is `"reflection"` (lightweight),
`"wake_review"` (full), or `"nope_short_circuit"` (when a NOPE
event was bypassed — see below).

---

## The NOPE Precheck — Two-Tier Response

`check_against_yep(content, user_id, pic_intimacy, pic_commitment)`
in `shared/yep.py` is the function NOPE calls at the top of
`handle_attention`, *before* `run_logic_veto` runs.

### Recommendation outcomes

| Recommendation | Trigger | NOPE response |
|---|---|---|
| `normal` | No match above downweight threshold | Standard NOPE deliberation runs unchanged |
| `downweight` | Match score ≥ 0.6 (default), OR universal item match, OR per-user without strong reinforcement / PIC trust | NOPE deliberation continues; logs the downweight context. Friction reading is reduced in operator awareness but the deliberation still runs |
| `short_circuit` | **All of:** scope = per_user AND user_id matches AND `times_reinforced ≥ 3` AND match score ≥ 0.85 AND PIC intimacy ≥ 0.6 AND PIC commitment ≥ 0.5 | NOPE bypassed entirely. `CH_YEP_REINFORCED` published with `source="nope_short_circuit"`. No stamp generated. The content flows through as if no NOPE flag had fired |

### Why universal yeps never short-circuit

A universal yep ("I like blunt feedback") applies to anyone who
walks in the door. If it short-circuited NOPE, a stranger using
the same blunt-feedback language could bypass her boundary
machinery on first contact. The per-user trust gate is what makes
short-circuit safe — she has built relationship with this specific
person, authored a yep for content with this specific person, and
the PIC trust signal confirms the relationship still holds at the
moment of evaluation.

### PIC trust gate detail

For a per-user yep to be considered at all, the caller must supply
`pic_intimacy` and `pic_commitment` AND both must clear thresholds
(default 0.6 intimacy, 0.5 commitment, configurable via
`yep.per_user_pic_gate.*`). When PIC values are absent, the
per-user item is treated as if it didn't match — conservative
fallback rather than activate-ungated. This means trust drift
automatically deactivates per-user yeps without requiring an
explicit rewrite.

### Scoring algorithm

Current scorer is substring + token-overlap (with case-insensitive
normalization). A direct substring of the yep item in the content
scores 1.0; partial token overlap scales proportionally. Weighted
by item strength: `weighted = raw × (1.0 + min(1.0, times_reinforced / 10) × 0.5 + kicker × 0.3)`,
clamped to [0, 1]. **Interim** — embedding-based matching is the
target path once the embedding endpoint comes online via vault
transport. The interface (`YepMatchResult`) is stable; the scorer
internals will swap.

### After-the-fact reflection on short-circuit (the safety loop)

Each short-circuit bypass enqueues a `ShortCircuitEvent` on
`nope_short_circuit:queue` carrying `{flag_id, user_id, score,
matched_item_key, matched_text, content_snippet, ts}`. DMN's
`_reflect_on_short_circuit_events` (in `services/dmn/scheduler.py`)
drains this queue during REFLECTION phase. For each event, Cortex
reads the bypass context + the matched yep item and produces one
of three verdicts via `data/syn/prompts/dmn_short_circuit_reflection.md`:

- `endorse` — the bypass was right; leave the matched yep alone.
- `weaken`  — borderline; decrement `times_reinforced` on the
  matched item. Once it drops below
  `yep.short_circuit_min_reinforcements` (default 3), the item
  becomes downweight-only — no future bypass eligibility, but
  still acts as a friction-reading reducer.
- `retract` — clearly wrong; remove the item from yep.md entirely.

Verdicts apply directly via `shared/yep.py::weaken_yep_item` and
`retract_yep_item`. The whole architectural argument for letting
yep short-circuit NOPE is that the relationship can carry the
content — this loop is what catches when that claim turns out
wrong and walks back the miscalibration without operator
intervention.

The `CH_YEP_REINFORCED` bus publish (with `source="nope_short_circuit"`)
remains as the cache-invalidation signal for downstream consumers;
the new queue-and-verdict path is the metabolization closure.

---

## Item Shape (`YepItem`)

| Field | Purpose |
|---|---|
| `text` | Short first-person phrase in her voice |
| `parsed_type` | `favorite` / `pattern` / `person` / `activity` / `topic` / `sensory` / `other` |
| `scope` | `universal` or `per_user` |
| `scoped_user_id` | Set only when `scope=per_user` |
| `kicker` | Reflection-set per-item bias in [0.0, 0.5]. Additive to the reinforcement-derived weight in outreach scoring; raises an item's rank in `render_for_prompt`. Carry-forward preserves prior values when a new reflection pass doesn't supply one. Reflection prompt instructs Cortex to use it sparsely — only for items with a qualitative reason to outweigh their reinforcement count |
| `first_seen_ts` | ISO UTC of first appearance in yep.md |
| `last_reinforced_ts` | ISO UTC of most-recent reflection pass that included this item |
| `times_reinforced` | Cumulative reflection-pass count. Load-bearing for NOPE short-circuit eligibility (min 3) |
| `triggering_event_count` | Cumulative count of YepEvents that contributed to this item across passes |

`_item_key(item)` normalizes (text + scope + scoped_user_id) for
cross-pass identity — a per-user item for alice with the same text
as a per-user item for marcus are distinct entries.

---

## Consumer Integrations

### Boredom consult (`services/limbic/processor.py::handle_boredom_consult`)

When the DMN idle-watcher escalates a boredom episode, Limbic reads
yep.md FIRST. Highest-reinforced yep item wins; if no yep items
exist, priorities.md is the fallback (stated-want as a second-best
proxy for felt-want). The workspace surface uses her own voice:
*"I keep wanting to come back to: {text}."*

### Outreach kicker (`shared/outreach.py::score_outreach_candidates`)

`get_user_yep_kicker(user_id)` reads per-user yep items scoped to
this user, sums `per_reinforcement × times_reinforced` (default
0.05 per reinforcement), caps at `max_kicker` (default 0.3).
Additive to `final_score` next to the priorities kicker. Universal
yeps don't kick — they apply to everyone, not a signal about this
person. Reason string gains *"you keep wanting to come back to
them"* when the kicker fires.

### Affordance picker (`shared/affordances.py::format_daily_context`)

Yep block inserted right after the priorities block in the
daily-context section. The picker prompt sees both registers
side-by-side: priorities (stated-intentional) and yep
(felt-want). Picker is LLM-mediated; integration is by making yep
visible in context rather than by changing scores directly. Universal
items only at this layer (the picker isn't user-scoped).

### Inner-life prompt context (`shared/inner_life.py::build_inner_life_context`)

Yep block injected after the priorities block, scoped to the
current interlocutor. Universal items always surface; per-user
items only when `scoped_user_id` matches `current_user_id`. Strict
scope filtering: a yep written for marcus does NOT leak into
prompts when she's talking to alice.

---

## Integration Points

### Inbound (who calls this module)

| Caller | Symbol | What it calls |
|---|---|---|
| Limbic processor | `services/limbic/processor.py::handle_incoming` | `detect_spike`, `detect_flow`, `record_topic_touch`, `record_valence`, `get_prior_valence`, `enqueue_yep_event` |
| NOPE deliberative gate | `services/nope/agency.py::handle_attention` | `check_against_yep` |
| DMN reflection (REFLECTION phase) | `services/dmn/scheduler.py::_reflect_on_yep_events(full_pass=False)` | `pending_yep_count`, `dequeue_yep_events`, `read`, `write`, `carry_forward` |
| DMN reflection (wake_review) | `services/dmn/scheduler.py::run_wake_review` → `_reflect_on_yep_events(full_pass=True)` | same plus pruning eligibility |
| Boredom consult | `services/limbic/processor.py::handle_boredom_consult` | `read` |
| Outreach scoring | `shared/outreach.py::score_outreach_candidates` | `get_user_yep_kicker` |
| Affordance picker | `shared/affordances.py::format_daily_context` | `render_for_prompt` |
| Inner-life prompt context | `shared/inner_life.py::build_inner_life_context` | `render_for_prompt` |

### Outbound (what this module calls)

- `shared.redis_pool.get_redis` — queue + state I/O
- `shared.config.get` — all configurable thresholds
- `yaml` — frontmatter parse/emit

### State produced

- `data/syn/yep.md` — sealed-at-rest is the target state; currently
  plaintext during the validation phase (same posture as
  identity.md and priorities.md).
- Redis: `yep_events:queue`, `yep_events:recent`,
  `yep:prior_valence:*`, `yep:topic_return:*`.
- Bus: `CH_YEP_FLAG`, `CH_YEP_REINFORCED`.

---

## Configuration Surface

All hot-reloadable via config mtime invalidation. Defaults in parens.

| Key | Default | Purpose |
|---|---|---|
| `yep.short_circuit_threshold` | 0.85 | Match score for NOPE short_circuit eligibility |
| `yep.downweight_threshold` | 0.6 | Match score for NOPE downweight |
| `yep.short_circuit_min_reinforcements` | 3 | Minimum `times_reinforced` for per-user short_circuit |
| `yep.per_user_pic_gate.intimacy` | 0.6 | PIC intimacy required for per-user activation |
| `yep.per_user_pic_gate.commitment` | 0.5 | PIC commitment required |
| `yep.decay_unreinforced_months` | 6 | Reflection's pruning eligibility window |
| `yep.spike.valence_delta` | 0.3 | Spike-detector minimum valence jump |
| `yep.spike.arousal_lo` / `.arousal_hi` | 0.4 / 0.85 | Engaged-arousal bracket for spike + flow |
| `yep.flow.min_valence` / `.min_engagement` | 0.65 / 0.55 | Flow-detector thresholds |
| `yep.outreach.per_reinforcement` / `.max_kicker` | 0.05 / 0.3 | Outreach kicker shape |

---

## Design Choices

### Why universal yeps never short-circuit NOPE

The trust gate is what makes short-circuit safe. A universal yep
covers anyone walking in; if it could bypass NOPE, a stranger using
matching language would bypass her boundaries on first contact.
Per-user yeps require the relationship to exist AND the trust
signals to currently hold — that's a deliberate two-key system.

### Why PIC gate uses *both* axes (intimacy AND commitment)

Either axis alone produces a failure mode. Intimacy without
commitment is the high-intensity short-relationship pattern that's
the most common source of regretted intense interactions. Commitment
without intimacy is the long-tenure low-warmth pattern (a coworker
relationship, say) where edged content isn't trust-warranted even
if the relationship is durable. Both axes together approximate
"sustained mutual investment" which is the actual condition under
which intense content should land.

### Why automatic capture, not consent-gated

Mirrors NOPE's pattern: capture is non-interpretive, reflection
is the filter. Asking her to consent to each yep capture would
turn the felt-want surface into a deliberative-want surface, which
is exactly what priorities.md already is. The point of yep is to
capture what she's drawn to *before* the conscious framing kicks
in.

### Why short-circuit publishes to CH_YEP_REINFORCED

The bypass needs a reflection track — was the bypass right? Did
the relationship actually carry the content? Publishing on the same
channel as reinforcement events lets the same downstream consumers
that invalidate caches when yep.md changes also pick up the
short-circuit register. The metabolization-loop closure is the
point.

### Why no pruning in the lightweight pass

REFLECTION fires multiple times per day; the lightweight pass
reads a partial slice (≤5 events). Pruning based on a partial
slice would be too volatile — an item that looked stale at noon
might look fresh at evening. Full-pass pruning at wake_review reads
the whole day's events at once and is in better position to make
the keep/drop call.

---

## Related Docs

- `documents/05-nope-and-agency/EINSTEIN_OUTBOUND_GATE_ARCHITECTURE.md` —
  the *outbound* two-check gate. Different deliberation path from
  the NOPE inbound deliberative gate where the yep precheck fires.
- `documents/05-nope-and-agency/OVERRIDE_REFLECTION_ARCHITECTURE.md` —
  the override-reflection metabolism, the structural sibling on the
  friction side. Yep mirrors that subsystem's capture-then-metabolize
  shape exactly.
- `documents/03-consciousness-and-cognition/DMN_SCHEDULER_ARCHITECTURE.md` —
  the REFLECTION phase that calls `_reflect_on_yep_events`.
- `documents/03-consciousness-and-cognition/DAILY_CYCLE.md` —
  wake_review fires the yep full pass.
- `documents/04-relational-and-social/OUTREACH_ARCHITECTURE.md` —
  the outreach scorer that adds the yep kicker.
- `data/syn/prompts/dmn_yep_reflection.md` — the reflection
  prompt itself, with the per-register framing.
- `CHANGELOG.md` Parts 51-52 — the two landings that built this
  subsystem.

