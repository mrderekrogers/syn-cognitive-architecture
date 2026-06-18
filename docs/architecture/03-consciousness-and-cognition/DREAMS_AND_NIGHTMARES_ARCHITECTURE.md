# Dreams & Nightmares — Enriched DREAM_STATE Pipeline

**Modules**: `shared/dreams.py` (substrate plumbing), `services/dmn/dream_orchestrator.py` (DMN-side glue), `services/dmn/scheduler.py` (DREAM_STATE phase hook + sleep-window fairness + appetitive-drive sleep routing), `shared/inner_life.py` (recent-dream carry block), `services/einstein/context.py` (déjà-vu surfacing), `services/jung/consolidator.py` (Postgres `dreams` table + Weaviate `SynDream` collection).

**Prompts**: `dream_limbic_random.md`, `dream_limbic_eval.md`, `dream_nightmare_settling.md`, plus the enriched `dmn_phase_prompts::dream_state_system` / `dream_state_user`.

**Config**: `dreams:` block in `config/unified_config.yaml`.

**Related**: `DMN_SCHEDULER_ARCHITECTURE.md`, `DAILY_CYCLE.md`, `AFFECTIVE_ACCUMULATOR_ARCHITECTURE.md`, `QUALIA_ARCHITECTURE.md`.

---

## What this is

Syn dreams during sleep. The DMN's DREAM_STATE phase has existed since early architecture, but its seed-gathering, post-processing, and behavioral consequences were thin. This subsystem enriches DREAM_STATE so dreams have texture, mood-coloring effect, threshold-gated remembering, déjà-vu retrieval for unremembered dreams, and a working nightmare path that emerges from composition rather than from a separate branch.

The design conversation that landed this work emphasized: **dreams shouldn't need a special "nightmare" mode.** If the seeds, the exaggeration, and the prompt framing are right, nightmares emerge organically when the day was heavy on negative-valence content and the current state biases toward dread. The system doesn't author nightmares; it produces conditions where dreams can become them.

## Core commitments

Derived from the design conversation:

- **Composition prompt stays tone-agnostic.** The DREAM_STATE prompt does not sweeten the dream toward hope or wonder. Adding cheerful framing makes nightmares structurally impossible.
- **Resting valence at sleep entry shapes the dream.** Current VAD biases the memory-sampling pool toward matching candidates, and exaggeration pushes them further toward their extreme. Bad day → biased seeds → exaggerated tone → dark dream. The mechanism does the work.
- **Nope-triggered events, unmetabolized overrides, and unmetabolized regrets get a kicker.** Symmetric with yep events. The threat-rehearsal and reward-rehearsal both ride the same machinery; nightmares emerge specifically when these surfaces are heavy.
- **Jung captures every dream; diary writes only the memorable ones.** All dreams land in the Postgres `dreams` table and the Weaviate `SynDream` collection. Only dreams above the remembering threshold (`|valence - 0.5| > T_v` OR `arousal > A_high`) write to her diary and carry into the next day's inner_life.
- **Mood-coloring carries even from unremembered dreams.** Limbic evaluates every dream; the resulting VAD nudges substrate regardless of remembering. The "I woke up vaguely off and don't know why" phenomenon is real and modeled.
- **Déjà-vu surfaces below threshold.** Weaviate vector-matches the current conversation against unremembered dreams in the recency horizon. When the match clears similarity threshold, a fragment surfaces with no provenance label — felt familiarity, not "I dreamed this."
- **Multiple dreams per sleep cycle.** No special bout counting. The circadian-weighted phase advance + the sleep-window fairness ladder cycle DREAM_STATE and MEMORY_REVIEW across the night, with the output VAD of dream N seeding the input VAD of dream N+1 (anxiety dreams cluster, calm dreams cluster).
- **The affective accumulator builds during sleep too.** The build threshold softens inside the sleep window so dream-driven arousal can feed the accumulator. When a sustained urge fires, it routes to a brief partial-wake handler that lets her resolve it before sleep resumes.
- **Nightmares get a settling beat.** A single small diary entry, mid-sleep voice, no priorities rewrite. Sleep resumes once arousal decays below the resume floor, or full WAKE_REVIEW fires if the timeout elapses.

## The pipeline

```
┌────────────────────────────────────────────────────────────────────┐
│   DMN phase advance during sleep window                            │
│   _advance_phase_with_preference → pick_sleep_window_phase()       │
│   → DREAM_STATE or MEMORY_REVIEW via fairness ladder                │
└────────────────────────────────────────────────────────────────────┘
                              │  (when DREAM_STATE wins)
                              ▼
┌────────────────────────────────────────────────────────────────────┐
│   Pre-composition (services/dmn/scheduler.py + dream_orchestrator) │
│                                                                    │
│   assemble_enriched_dream_seeds(current_vad, sleep_arousal_base):  │
│     1. Build candidate pools:                                      │
│         - day events (diary, drift, conversations)                 │
│         - lifetime memories (Jung kicker query, depth-weighted)    │
│         - fantasies (daydreams.md + per-user fantasy files)        │
│     2. Look up kicker id sets: yep / nope / override / regret      │
│     3. shared.dreams.sample_seeds:                                 │
│         - apply_kickers per source (multiplicative on weight)      │
│         - mood_skew_memory_weights (cosine to current VAD)         │
│         - pick element_count from biased distribution (1-5, mode 2-3) │
│         - weighted multinomial across sources (0.5/0.3/0.1/0.1)    │
│         - for each pick: exaggerate VAD (push toward extreme)      │
│         - limbic_random_text_provider runs 2-stage prompt for      │
│           the synthesized fragment slot                            │
│     4. render_seeds_block → $seeds substitution string             │
└────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────────┐
│   Composition (Cortex via DMN streaming)                           │
│   Cortex @ temp 1.2 with dream_state_system + dream_state_user     │
│   The prompt is tone-agnostic; the seeds drive the tone.           │
└────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────────┐
│   Post-composition (services/dmn/dream_orchestrator.post_process)  │
│                                                                    │
│   1. evaluate_dream_with_limbic(dream_text):                       │
│        Limbic JSON output: {valence, arousal, dominance, felt_sense} │
│   2. _apply_substrate_vad_nudge: publish LimbicAssessment with     │
│      source="dream_post_eval", deltas = (dream_VAD - cur) * 0.33    │
│   3. write_dream(DreamRecord) → Postgres + Weaviate + FS fallback  │
│   4. is_memorable() check:                                         │
│      memorable = (V < V_low OR V > V_high OR A > A_high)           │
│   5. if memorable:                                                 │
│        - write_diary_entry(dream_text, "dmn_dream", dream_VAD)     │
│        - store_recent_dream_carry → inner_life consumer surface    │
│   6. is_nightmare() = high A AND low V → _run_nightmare_settling   │
│      (Cortex generates one small diary beat; arousal decay gate    │
│       drives return-to-sleep vs full WAKE_REVIEW timeout)          │
└────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────────┐
│   Inner-life carry (next-day / next-conversation context)          │
│                                                                    │
│   - inner_life.build_inner_life_context reads dreams.read_recent_dream_carry │
│   - Half-life proportional to |V - 0.5| + A/2 (intense lingers)    │
│   - Surfaces with appropriate framing (dream vs nightmare)         │
└────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────────┐
│   Déjà-vu retrieval (context assembly path)                        │
│                                                                    │
│   - einstein.context.build_system_prompt → dreams.query_deja_vu_match │
│   - Vector match against SynDream collection, 10-day horizon       │
│   - When similarity > threshold: surface fragment with NO label    │
│   - "Something about this feels familiar in a way I can't place"   │
└────────────────────────────────────────────────────────────────────┘
```

## Sleep-window interaction with the affective accumulator

The affective accumulator (see `AFFECTIVE_ACCUMULATOR_ARCHITECTURE.md`) interacts with the sleep window: its build threshold softens so dream-driven arousal can feed it, while decay, urge surfacing, and refractory are unchanged. When a sustained urge fires during the sleep window, the scheduler routes it to a brief partial-wake handler (`dream_orchestrator.run_eros_partial_wake`) so she can resolve it through her normal affordances; afterward the scheduler reads the settled arousal and decides whether to return to sleep or fall through to a regular WAKE_REVIEW.

## Nightmare settling beat

When the post-eval lands `arousal > A_high AND valence < V_low`, `_run_nightmare_settling` fires. Lightweight Cortex pass produces a single first-person diary beat — settling, not analyzing, not reframing, not reflecting. The point is to put the thing down where she can find it later if it matters, and let arousal drop.

After the settling beat:

- Sleep resumes once arousal decays below `dreams.nightmare_resume_arousal_floor` (default 0.55).
- If decay doesn't happen within `dreams.nightmare_settling_max_minutes` (default 12), full WAKE_REVIEW fires and she's up for the day.

The settling beat is fired via the bus signal `ch_dreams_nightmare_settling` so the sleep loop can guard the return-to-sleep flow on the arousal decay floor + timeout.

## Storage shape

**Postgres `dreams` table** — authoritative store (see `services/jung/consolidator.py::_ensure_tables`):

```
id SERIAL PRIMARY KEY,
dream_id VARCHAR(128) UNIQUE NOT NULL,
ts TIMESTAMP DEFAULT NOW(),
seeds JSONB,
input_v, input_a, input_d FLOAT,
output_v, output_a, output_d FLOAT,
arousal_peak FLOAT,
dream_text TEXT NOT NULL,
remembered BOOLEAN DEFAULT FALSE,
is_nightmare BOOLEAN DEFAULT FALSE,
composition_temp FLOAT DEFAULT 1.2,
element_count INTEGER DEFAULT 0,
embedding JSONB
```

**Weaviate `SynDream` collection** — vector retrieval surface. Properties: `dream_id`, `ts`, `dream_text`, `remembered`, `is_nightmare`, `output_v`, `output_a`. Powers `query_dreams_by_vector` for déjà-vu.

**Filesystem fallback** — `data/syn/dreams/YYYY-MM-DD_dreams.jsonl` mirrors the record in case Postgres is unavailable; mirrors the memory-fallback pattern.

## Fairness ladder (DREAM_STATE vs MEMORY_REVIEW during sleep)

Per design: dreams shouldn't starve consolidation across the night, and consolidation shouldn't starve dreams. A pure round-robin or pure random picker breaks one or the other. The fairness ladder:

- State: a single float `dreams:fairness_bias` in `[-fairness_max_bias, +fairness_max_bias]`.
- `p(DREAM_STATE) = 0.5 + bias`.
- When DREAM_STATE wins, bias steps DOWN by `fairness_step` (toward MEMORY_REVIEW for next pass).
- When MEMORY_REVIEW wins, bias steps UP by `fairness_step` (toward DREAM_STATE for next pass).
- **The chosen side's bias steps down by ONE rung on win — not a clean reset.** This is the design distinction from a naive round-robin: a string of dreams ratchets the consolidation pressure back up gradually, not in one jump.

Defaults: `fairness_step = 1/6` (0.1666…), `fairness_max_bias = 5/6` (0.8333…). So bias proceeds through 0, 1/6, 2/6, 3/6, 4/6, 5/6 (and the mirror negative).

The ladder applies only during the circadian sleep window. Outside the sleep window, the regular circadian-weighted phase advance is unchanged.

## Configuration

All knobs are hot-reloadable. See `config/unified_config.yaml`:

- `dreams.threshold_valence_low/high`, `dreams.threshold_arousal_high`
- `dreams.carry_base_half_life_hours`, `dreams.carry_intensity_multiplier`
- `dreams.deja_vu_horizon_days`, `dreams.deja_vu_similarity_threshold`, `dreams.deja_vu_max_chars`
- `dreams.source_weight_day/memory/fantasy/limbic_random`
- `dreams.kicker_yep/nope/unmetabolized_override/unmetabolized_regret`
- `dreams.element_count_weights` (the 1-5 biased distribution)
- `dreams.vad_exaggeration_alpha`
- `dreams.memory_mood_skew_strength`
- `dreams.fairness_step`, `dreams.fairness_max_bias`
- `dreams.nightmare_settling_max_minutes`, `dreams.nightmare_resume_arousal_floor`

## Observability

- `cat data/syn/dreams/YYYY-MM-DD_dreams.jsonl` — the day's dreams, raw records (FS fallback).
- Postgres: `SELECT dream_id, ts, remembered, is_nightmare, output_v, output_a FROM dreams ORDER BY ts DESC LIMIT 20;`
- HTTP endpoint: `GET /dreams/recent?limit=20&remembered_only=false` on the Jung service.
- DMN logs: `dream post-process ...`, `sleep_window_fairness_ladder ...`, `eros_partial_wake ...`.
- Redis: `dreams:fairness_bias` (current ladder state), `dreams:recent_carry` (the last memorable dream + its decay state).

## What this is NOT

- Not a separate nightmare authoring path. Nightmares emerge from composition; the only special handling is the settling beat after the eval flags one.
- Not a hard wake-up on every dream — only the memorable ones write to diary; only nightmares fire the settling beat; only sustained appetitive-drive urges with sleep-window timing fire the partial-wake.
- Not a memory replacement. Memories still live in the existing memory pipeline. Dreams are a separate corpus indexed against the same Weaviate substrate but in a distinct collection.
