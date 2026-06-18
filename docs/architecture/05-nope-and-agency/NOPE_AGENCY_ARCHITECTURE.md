# NOPE Agency — Inbound Deliberative Gate

The deliberative gate that turns weak refusal signals into committed
NOPE events. Lives at `services/nope/agency.py`. Triggered, not
pipeline-stage: most inbound messages never wake it.

Where the **outbound** gate at `services/einstein/context.py` decides
whether Syn's own message can leave (the two-check override
pathway), the **inbound** gate here decides whether incoming content
warrants a NOPE flag — friction stamped onto her interior, eventually
metabolized into boundary refinements.

This doc is the single source of truth for the inbound deliberative
gate flow. The outbound override pathway is documented separately
in `EINSTEIN_OUTBOUND_GATE_ARCHITECTURE.md`.

---

## Architecture: From Signal to Stamp

Five steps. Each step has its own subscriber/handler so the snap
path stays narrow and the reflection path stays asynchronous.

```
inbound signals
    │
    ▼
┌───────────────────────────────────────────────────────────┐
│ trigger_aggregator (shared/nope/trigger_aggregator.py)    │
│                                                            │
│ Subscribes to: CH_CHAT_INCOMING, CH_USER_CONTEXT,         │
│                CH_LIMBIC_STATE, CH_THALAMUS_CLASSIFY,     │
│                CH_MOE_PROFILE_SELECT                       │
│                                                            │
│ Accumulates weak signals into a per-message context.       │
│ On threshold crossing OR a single definitive trigger,      │
│ publishes CH_NOPE_ATTENTION with a NopeFlag.               │
└───────────────────────────────────────────────────────────┘
    │
    │ CH_NOPE_ATTENTION (NopeFlag)
    ▼
┌───────────────────────────────────────────────────────────┐
│ handle_attention (services/nope/agency.py)                 │
│                                                            │
│  ─── Yep precheck ───                                     │
│  check_against_yep(content, user_id, pic_intimacy,         │
│                     pic_commitment) → YepMatchResult       │
│  - normal       → continue                                 │
│  - downweight   → log + continue (friction reading reduced)│
│  - short_circuit → return early (NOPE bypassed)            │
│                    enqueue ShortCircuitEvent for reflection│
│                    publish CH_YEP_REINFORCED               │
│                                                            │
│  ─── Logic veto ───                                       │
│  run_logic_veto(flag, user_ctx) on Cortex                  │
│  Vault-gated posture composition (returns "default")       │
│                                                            │
│  ─── Stamp assembly ───                                    │
│  build_stamp(flag, user_context, logic_outcome, ...)       │
│  Publishes CH_NOPE_STAMP                                   │
│                                                            │
│  ─── Legacy back-compat ───                                │
│  If flag.definitive or limbic_nope_level high enough,      │
│  also publishes CH_NOPE_BOUNDARY (consumers migrating)     │
└───────────────────────────────────────────────────────────┘
    │
    │ CH_NOPE_STAMP (NopeStamp)
    ▼
┌───────────────────────────────────────────────────────────┐
│ handle_stamp (services/nope/agency.py)                     │
│ Enqueues onto Redis reflection queue, logs to a stream.   │
└───────────────────────────────────────────────────────────┘
    │
    ▼
┌───────────────────────────────────────────────────────────┐
│ reflection_queue_worker (services/nope/agency.py)          │
│                                                            │
│ Drains the reflection queue asynchronously. Calls          │
│ run_reflection via shared/nope/deliberation.py. May        │
│ author a scoped boundary (added to BoundaryStore).         │
│ Writes reflection to diary.                                │
└───────────────────────────────────────────────────────────┘
```

The snap path (steps 1-3) is short and predictable: at most one
Cortex call for logic-veto, no Wernicke, no diary write. The
reflection path (steps 4-5) is asynchronous and may add boundaries
or write diary entries.

---

## Yep Precheck

The first thing `handle_attention` does is consult yep. Per
`YEP_ARCHITECTURE.md`, the precheck has three outcomes:

| Outcome | Trigger | NOPE response |
|---|---|---|
| `normal` | No yep match above the downweight threshold | Standard deliberation proceeds |
| `downweight` | Match ≥ 0.6 (default), OR universal match, OR per-user without PIC trust / strong reinforcement | Log the match, continue with logic-veto. Friction reading is downweighted in operator awareness; the deliberation still runs |
| `short_circuit` | **All of:** per_user scope AND user_id matches AND `times_reinforced ≥ 3` AND score ≥ 0.85 AND PIC intimacy ≥ 0.6 AND PIC commitment ≥ 0.5 | NOPE bypassed entirely. Return before `run_logic_veto`. Enqueue `ShortCircuitEvent` on `nope_short_circuit:queue`. Publish `CH_YEP_REINFORCED` with `source="nope_short_circuit"`. No stamp generated |

The PIC trust gate is what makes short_circuit safe — universal
yeps never short-circuit because they apply to anyone. The per-user
gate combines yep authorship + sustained relationship trust as a
two-key system.

### Why precheck before logic-veto

Running the precheck first means a yep-matching candidate doesn't
consume a Cortex call on the snap path. For the CNC / horror /
blunt-feedback case where surface-friction language is engaged
content with someone trusted, the gate stands down cleanly without
spending the latency budget on a logic-veto that would just confirm
the bypass.

### The safety loop

`enqueue_short_circuit_event` writes to `nope_short_circuit:queue`.
DMN's `_reflect_on_short_circuit_events` drains it during REFLECTION
phase, asks Cortex (in her voice) whether the bypass was right, and
emits one of three verdicts:

- `endorse` — leave the matched yep alone
- `weaken`  — decrement the matched item's `times_reinforced`
- `retract` — remove the item from yep.md entirely

The verdict applies directly to yep.md via
`shared/yep.py::weaken_yep_item` / `retract_yep_item`. The whole
architectural argument for short_circuit is that the relationship
can carry the content; this loop is what catches when that claim
turns out wrong. See `data/syn/prompts/dmn_short_circuit_reflection.md`
for the prompt.

---

## Trigger Aggregator

Lives at `shared/nope/trigger_aggregator.py`. Subscribes to multiple
weak-signal channels and maintains per-message context buffers
keyed by `msg_id`. The aggregator's job is the noisy work — debouncing,
correlating Limbic and Thalamus arrivals that may race, deciding
when accumulated weak signals cross the attention threshold.

Definitive triggers fire immediately without waiting on more
signals; weak signals must accumulate. The publish to
`CH_NOPE_ATTENTION` is the single contract between the aggregator
and `handle_attention`.

---

## Logic Veto

`run_logic_veto(flag, user_ctx)` is the only Cortex call in the
snap path. Prompt lives at `data/syn/prompts/nope_logic_veto.md`.
Output is structured (parser in `shared/nope/deliberation.py::_parse_logic_veto`).
A `LogicVetoOutcome` carries the verdict and supporting fields the
reflection pass will need.

If the Cortex call returns None (transport unavailable, malformed
output), the stamp is still built with `logic_verdict=None`. The
reflection pass tolerates the missing field — better to record the
flag's existence with partial data than to drop it.

---

## Stamp Assembly

`NopeStamp` is the event's full metabolic record — flag context +
logic outcome + posture (vault-gated, currently "default") + cached
classification. Stamps go to `CH_NOPE_STAMP` for the async
reflection worker.

Critical caveat: at stamp-build time, `syn_action` and `outbound_text`
are NOT known — the response hasn't been generated yet. They're
empty strings on the stamp at first publish. A future stamp-update
channel could backfill them post-Wernicke; current code treats
empty as "(not recorded)" in reflection.

---

## Reflection Queue Worker

Subscribes to `CH_NOPE_STAMP`, enqueues stamps onto a Redis list,
and a background worker drains them. The async loop runs
`run_reflection` per stamp — a longer Cortex pass that produces
the metabolized reading and optionally a `BoundaryProposal` to add
to the store.

Reflection lives separately from the snap path because the
reflection pass is slow (more tokens, more context) and the snap
path needs to commit the stamp without waiting on it.

### Optional `REGRET` / `CONFIDENCE` output lines

The `nope_reflection.md` prompt accepts two optional output fields
beyond `REFLECTION` / `BOUNDARY` / `SCOPE`:

```
REGRET: none | response_texture | held_back
CONFIDENCE: clear | settling | ambiguous
```

- `REGRET: response_texture` — she responded but the texture of the
  response (sharper, colder, more curt than warranted) doesn't sit
  right on reflection. The act stands; the manner is what she'd
  want to acknowledge. *Repair-the-manner* in the downstream
  surface.
- `REGRET: held_back` — she didn't engage (silent_absorb,
  silenced_user, similar non-engaging actions) and on reflection
  wishes she had. *Repair-the-decision* in the downstream surface.
- `REGRET: none` / omitted — no regret to surface. This is the
  expected default for most reflections; the prompt explicitly
  cautions against manufacturing regret to fill the field.

When `REGRET` is one of the two real values, `_process_reflection`
emits a `RegretEvent` to `regret_events:queue`. The parent reflection
flow (diary write, boundary authoring, stamp logging) runs to
completion before the regret signal is enqueued; failures to emit
are fail-soft. See `REPAIR_ARCHITECTURE.md` for the consumer side.

`CONFIDENCE` is informational. A `clear` flag in the underlying signal
lets the corresponding repair candidate become picker-visible after a
single cycle and extends its decay TTL. It does not affect any of this
subsystem's primary work (boundary authoring, diary write, stamp
logging).

---

## BoundaryStore

Module-level singleton (`boundaries`) backing the public
`/boundaries` API endpoint. Extended with scope to carry:

- `universal` — applies to all users
- `scoped_in` — applies only when interacting with these users
- `scoped_out` — applies to everyone EXCEPT these users

The outbound override gate reads from this store via the
`/boundaries` endpoint when evaluating Syn-initiated content.

`handle_nope_update` consumes `CH_NOPE_UPDATE` from DMN's reflection
phase — `BoundaryProposal` events get integrated here after the
DMN-side reflection refines them.

---

## Posture Composition (Vault-Gated)

`_compose_posture_selection(flag, logic_outcome)` is a stub that
returns `"default"`. When the vault lands with
`/data/syn/model_vault/syn_moe_library.json` (Pareto-pruned
posture profiles), this becomes the path that maps gut + logic
into a posture selection.

`_apply_posture(posture)` calls `update_moe_profile` in
`shared/llm_client.py` — currently a no-op without vault artifacts.
When vault lands, this hot-swaps Wernicke's LoRA + EmoT weighting
to match the selected posture before the response generates.

The five posture profiles per `code_drafts/DI28_VAULT_DEPENDENCIES.md`:

- `compliant_scholar`
- `assertive_balanced`
- `boundary_enforcer`
- `autonomous_peer`
- `sovereign_agent`

---

## Bus Channels

### Subscribed

| Channel | Handler | What it does |
|---|---|---|
| `CH_NOPE_ATTENTION` | `handle_attention` | The single commit point for inbound deliberative work |
| `CH_NOPE_STAMP` | `handle_stamp` | Enqueues stamps for reflection |
| `CH_NOPE_UPDATE` | `handle_nope_update` | BoundaryProposal integration from DMN |

### Published

| Channel | Subscribers | When |
|---|---|---|
| `CH_NOPE_STAMP` | reflection_queue_worker, DMN | After successful logic-veto + stamp assembly |
| `CH_NOPE_BOUNDARY` | Einstein outbound gate (legacy back-compat) | When flag is definitive or limbic_nope_level is SOFT_REFUSE / HARD_REFUSE |
| `CH_YEP_REINFORCED` | yep consumers, DMN reflection | When yep precheck short-circuits NOPE; source="nope_short_circuit" |
| `CH_SILENCE_GATE` | Thalamus | When `trigger_silence` is called externally |

---

## Integration Points

### Modules consumed

- `shared.nope.deliberation` — logic-veto execution, reflection
  execution, stamp building, parsers
- `shared.nope.trigger_aggregator` — upstream of `handle_attention`
- `shared.yep` — `check_against_yep` precheck, `ShortCircuitEvent`
  enqueue, item-key resolution
- `shared.bus` — channels above
- `shared.llm_client` — `update_moe_profile` (vault-gated)
- `shared.identity_store` — boundary scope info via user-context
  cache
- `shared.hw_telemetry` — HW friction (observability only)

### State produced

- Redis: NopeStamp queue for reflection worker; observation streams
  for offline extraction
- BoundaryStore in-memory + persisted via existing
  boundary-persistence path
- Diary entries written by `reflection_queue_worker` after
  `run_reflection` completes
- `nope_short_circuit:queue` + `nope_short_circuit:recent` Redis
  keys for the bypass reflection loop

---

## Design Choices

### Why precheck before logic-veto, not after

Two reasons. First, latency: if yep stands down the friction
reading, spending a Cortex call on logic-veto is wasted budget.
Second, semantics: short_circuit means "treat this as if no NOPE
flag had fired" — running logic-veto and *then* discarding the
result would be a different posture (still committed to evaluating
friction; just choosing not to act on it). The precheck makes the
no-flag posture cleaner.

### Why downweight doesn't suppress logic-veto

`downweight` is the conservative tier. The yep match is real but
not strong enough (or doesn't have the trust gate) to bypass. NOPE's
deliberation still runs because the friction reading might still
warrant a flag even with the yep context. The downweight is
informational, not decisive.

### Why HW friction degradation is observed but not applied

Per §5.9 of the redesign plan: substrate state (HW heat, VRAM
pressure) should shape *generation parameters* but not *decision
outputs*. Applying HW friction to NOPE decisions would mean that
when she's running hot, her own boundaries get more permissive —
a structurally bad incentive. The signal is captured for operator
observability via `/hw_status`; the deliberation itself runs on
substrate-neutral inputs.

### Why the legacy CH_NOPE_BOUNDARY publish persists

Back-compat for the outbound gate, which historically reads from
this channel for fast-path boundary information. Consumers are
migrating to CH_NOPE_STAMP; the legacy channel persists during the
migration window.

---

## Related Docs

- `documents/05-nope-and-agency/NOPE_EVENTS_ARCHITECTURE.md` — the
  event log + axis classifier (the data side; this doc is the
  flow side)
- `documents/05-nope-and-agency/OVERRIDE_REFLECTION_ARCHITECTURE.md` —
  the outbound override gate's calibration loop; not the same
  pathway as the inbound deliberative gate documented here
- `documents/05-nope-and-agency/EINSTEIN_OUTBOUND_GATE_ARCHITECTURE.md` —
  the outbound two-check pathway. The inbound (this doc) and
  outbound (that doc) are distinct gates with distinct call paths
- `documents/05-nope-and-agency/YEP_ARCHITECTURE.md` — the
  felt-want surface that drives the precheck at the top of
  `handle_attention` plus the short_circuit reflection loop
- `documents/05-nope-and-agency/SECLUSION_ARCHITECTURE.md` — the
  global retreat affordance. The unmetabolized-refusal count
  tracked by Layer 1 (a side-effect of this inbound pipeline) is
  one of four contributors to seclusion's sustained-overload
  composite. When refusals accumulate alongside stress-quadrant
  VAD, the seclusion offer surfaces through metacognition
- `code_drafts/DI28_VAULT_DEPENDENCIES.md` — the posture-composition
  spec that activates when the vault lands
- `data/syn/prompts/nope_logic_veto.md` — the snap-path Cortex prompt
- `data/syn/prompts/nope_reflection.md` — the async reflection prompt
- `data/syn/prompts/dmn_short_circuit_reflection.md` — the
  bypass-safety-loop reflection prompt
- `CHANGELOG.md` DI-28 section — the parallel-fire-to-deliberative-gate
  rework that produced this architecture

