# Two-Layer Mood — Affective "Now" + Slower Cognitive VAD

**Status:** IMPLEMENTED 2026-06-06 (laptop + both nodes). See `DESIGN_DECISIONS.md §X` and the CHANGELOG.
**Author:** mapping pass, 2026-06-06.

**As-built deltas from the original map below:**
- The cognitive layer tracks the affective **excursion around its own rest**, not the absolute value — absolute-tracking collapsed both layers at idle (caught by the §5 sim). This is the load-bearing change vs. the first draft of §2.2/§2.3.
- Final tuning (per the sim): `pull_from_affective=0.15`, `affective_anchor_pull=0.12`, `compare_interval_sec=12`, cognitive `decay_duration=3600` (1 h, per D — not the 30 min first proposed).
- Affective rest is `[0.5, 0.4, 0.2]` and cognitive rest `[0.55, 0.45, 0.65]` (per D); the affective rest is applied **everywhere** `vad_baseline` is used (decay + `scale_for_model` + seclusion).
- Cognitive steering regions: `cortex` **and** `thalamus`.
- DMN: no code change — its VAD use (dreams/diary/bleed/boredom/outbound) stays affective by design; DMN-scheduled Cortex thoughts ride the cognitive layer via the cortex-region steering.
**Problem statement (D):** Syn's mood is too easily knocked around by individual messages. Keep the current mood as the *top affective layer* ("the Now," the affective-node side), but add a *slower cognitive layer* that does not move directly off any single message — it is pulled by the current mood, stabilizes faster toward its own rest, and at the same comparison beat pulls the affective side back toward it so the two always try to meet in the middle. The cognitive layer is what Thalamus and Cortex read. Each layer rests at a *different* point so they genuinely diverge.

---

## 1. How it works today (mapped)

### 1.1 The single live mood: `vad_global`

All of Syn's live mood lives in **one** vector, `Layer1State.vad_global`, in `services/layer1/picvad.py` (runs as the `layer1` service on **Node 2**, the affective hemisphere).

- **Moved directly, per message,** by `process_limbic()` (picvad.py:1453):
  ```
  valence   = 0.6*old + 0.4*reaction
  arousal   = max(0.5*old, reaction)      # arousal jumps, never averaged down
  dominance = 0.7*old + 0.3*reaction
  ```
  Plus direct spikes from nope refusals (picvad.py:1908, +0.5 arousal), appetitive-accumulator discharge, basal drift, and reach-failure disruption.
- **Decays toward `vad_baseline = (0.5, 0.5, 0.5)`** every 1 s tick via `decay_vad()` (picvad.py:519), exponential half-life `emotional_physics.vad.decay_duration = 10800 s` (3 h). An unmetabolized refusal slashes *arousal* decay to 5 % (the metabolization lock).
- **Fatigue drag on the rest target** (`_effective_vad_baseline`, picvad.py): once accumulated sleep pressure passes its limit, tiredness pulls the rest target down so she settles instead of staying keyed-up. Two components, both refreshed every 30 s from sleep pressure: an **arousal drop** (`fatigue_arousal_drop`, §M 2026-06-04 — active energy winds down) and, as of **2026-06-07**, a **valence drop** (`fatigue_valence_drop` — being overtired dulls her mood). The valence drag uses a gentler coeff/cap (`sleep_pressure.valence_drag_*`: limit 0.8, coeff 0.35, cap 0.2) than the arousal drag so tiredness *flattens affect* without manufacturing real sadness; both are floored at 0.1. This is the substrate counterpart to the "tiredness can hinder my mood" lines in `cognitive_state_injection.md`, so what she's told she might feel and what the rest target does agree.
- **`compute_effective_vad()`** (picvad.py:683) blends `vad_global` with the per-user relational VAD (0.4/0.6 on valence/dominance, `max` on arousal) → `vad_effective`.
- Every tick, `vad_effective` is resolved into a **named emotion + per-model capacity** by `model_emotion_profile.get_profiler().resolve_mood()` and into **emotion-LoRA steering** per region.

### 1.2 Who consumes the mood

| Consumer | Node | Reads | Path |
|---|---|---|---|
| Wernicke (the felt voice) | 2 | `vad_scaled` (= `scale_for_model(vad_effective,"wernicke")`) + named mood | Einstein `context.py:631` → `/state/wernicke` |
| Emotion-LoRA steering (wernicke **and cortex** regions) | 1+2 | `vad_effective` for *all* regions | `tick()` per-region loop, picvad.py:951 |
| Cortex deliberation | 1 | model capacity for the cortex model; cortex region steering | profiler + LoRA |
| Thalamus routing | 1 | identity + cognitive_state (novelty/load/somatic); arousal indirectly | router.py |
| DMN (dreams/scheduling) | 1 | pins + arousal/intimacy saturation | `/state`, `/pins` |
| Mona dashboard | 1+2 | `vad_effective` sampled every 15 s into the rhythm panel | app.py:5403 |

**The core issue:** one fast vector feeds *both* the in-the-moment felt voice (correct — it should be reactive) *and* the deliberative side (wrong — Thalamus/Cortex inherit every per-message jerk).

### 1.3 Deployment topology

- **Node 1 (Analytical):** cortex, thalamus-router, thalamus-llm, dmn, jung, freud, lora, embedder, mona, redis, postgres, weaviate, searxng.
- **Node 2 (Affective):** wernicke, **layer1 (picvad)**, limbic, einstein, nope, hands/toolbelt, teleport, freud, mona, redis, postgres.
- Code deployed to both nodes. Laptop holds the source of truth; `sync_edit.py` (paramiko over SSH) pushes; `scripts/start_node{1,2}.sh` start each hemisphere. `config/unified_config.yaml` must be byte-identical on both nodes.

---

## 2. Proposed design

### 2.1 Two vectors, two roles

| Layer | Vector | Role | Moves from | Rests at | Read by |
|---|---|---|---|---|---|
| **A — Affective "Now"** | `vad_global` (unchanged) | Surface mood, the felt instant | Each message / spike (limbic, nope, appetitive drive, drift), as today | `affective` rest `(0.50, 0.50, 0.50)` | **Wernicke**, wernicke-region steering, Mona "Now", pins, seclusion, appetitive drive |
| **B — Cognitive mood** | `vad_cognitive` (**new**) | Composed, deliberative backdrop | *Never* from a single message — only the low-pass pull from Layer A | `cognitive` rest `(0.55, 0.45, 0.60)` (divergent) | **Thalamus, Cortex**, cortex-region steering, DMN scheduling |

Layer A keeps Syn's *expression* responsive (a sharp message still lands in the voice). Layer B gives her *thinking* a steadier emotional floor that a single message can't yank around.

### 2.2 The three couplings (run on the comparison beat)

At each **comparison tick** (every `compare_interval_sec`, proposed **12 s** — slower than the 1 Hz affective tick, aligned to Mona's 15 s sampler), with `dt` = seconds since the last comparison:

1. **Cognitive self-decay toward its own rest** — *faster* time-constant than affective ("stabilizes faster"):
   ```
   f_c = exp(-ln2 * dt / cognitive.decay_duration)        # decay_duration = 1800 s (30 min)
   vad_cognitive = C_rest + (vad_cognitive - C_rest) * f_c
   ```
2. **Cognitive pulled by the current mood** (low-pass of Layer A — this is the only thing that moves it off rest):
   ```
   vad_cognitive += cognitive.pull_from_affective * (vad_global - vad_cognitive)   # ~0.50
   ```
3. **Affective pulled gently back toward cognitive** (the stabilizer — damps the per-message jerk; deliberately small so the voice stays reactive):
   ```
   vad_global    += cognitive.affective_anchor_pull * (vad_cognitive - vad_global)  # ~0.08
   ```

Layer A's existing 1 Hz self-decay toward `(0.5,0.5,0.5)` and all message injection stay exactly as they are. The new code only adds: cognitive self-decay, the two cross-pulls, and a 12 s gate.

### 2.3 Why the rests must diverge, and where they "meet"

If both layers rested at `(0.5,0.5,0.5)`, the cognitive layer would be a redundant smoothed copy and there'd be nothing to diverge to at idle. Distinct rests guarantee a *standing offset*: the cognitive read is, by construction, a touch calmer (arousal 0.45 < 0.50), a touch more disposed-positive (valence 0.55), and more composed/in-control (dominance 0.60) than raw affect — "thinking feels steadier than feeling." Tunable; the point is **A_rest ≠ C_rest**.

Because pulls are **gap-proportional**, the system self-balances:

- **Under load** (affective spikes far from cognitive): the cross-pull terms are large → cognitive tracks the smoothed mood and reins the affective spike in. They *converge toward the middle*.
- **At idle** (both near their rests, mutual gap ≈ `|A_rest − C_rest|` ≈ 0.05–0.10): self-decay dominates the small cross-pull → each settles near its *own* rest. They *diverge*.

The joint idle fixed point (per axis), letting `p` = `pull_from_affective`, `q` = `affective_anchor_pull`, and `d_a`,`d_c` the per-beat self-decay fractions toward each rest, is the solution of the 2×2 linear system — in practice cognitive lands just below its rest and affective just above neutral, separated by a stable gap. Keep `q` small and `cognitive.decay_duration` short enough that idle divergence ≈ the rest separation; that is the single most important tuning relationship and should be verified with the simulation in §5.

### 2.4 Emotion-LoRA / profiler routing (the crux of "Thalamus and Cortex use it")

Today `tick()` interpolates steering for *every* region from `vad_eff`. Change it to be **source-aware per region**:

- **wernicke region** ← `vad_effective` (affective, unchanged).
- **cortex region** ← a cognitive effective VAD (`vad_cognitive` composed through the same relational blend, or `vad_cognitive` directly — recommend directly, since cognition is less relationship-reactive).

Likewise resolve the profiler **twice**: an *affective mood* (for Wernicke + Mona "Now") and a *cognitive mood* (for Cortex capacity + Mona "Backdrop"). This is the cleanest place the split actually changes behavior, and the trickiest diff — it touches the per-region loop and the `EmotionalState` payload.

---

## 3. Implementation map (files, by node)

### Node 2 (affective) — the engine
1. **`shared/models.py`** — add `vad_cognitive: VADVector` (and optional `mood_cognitive_*` fields) to `EmotionalState`.
2. **`shared/vad_math.py`** — add `vad_cognitive_neutral()` / `_VAD_COGNITIVE_NEUTRAL` constant (the divergent rest).
3. **`services/layer1/picvad.py`** — the bulk:
   - `Layer1State.__init__`: read `emotional_physics.vad.cognitive.*`; init `self.vad_cognitive`, `self.vad_cognitive_baseline`, `self._last_cognitive_compare`.
   - New `compare_cognitive(dt)` method implementing §2.2 (three couplings).
   - `tick()`: after the affective decay, if `now - _last_cognitive_compare >= compare_interval_sec`, call `compare_cognitive()`; resolve the cognitive mood; make the per-region steering loop source-aware (§2.4); add `vad_cognitive` + cognitive mood to the returned `EmotionalState`.
   - Endpoints: include `vad_cognitive`/cognitive mood in `/state`; add `/state/cognitive/{model}` and `/mood/cognitive` mirrors of the existing affective ones.
   - Decide fatigue-drag application to the cognitive arousal rest (recommend: yes, same `_fatigue_arousal_drop`).
4. **`services/einstein/context.py`** — Wernicke prompt path stays on `vad_scaled` (affective). No change required for Wernicke; only relevant if you also want the cognitive named-mood surfaced in any cortex-bound substrate.

### Node 1 (analytical) — the consumers
5. **`services/thalamus/router.py`** — where routing consults arousal/mood, read the cognitive layer (`/state` `vad_cognitive` or `/state/cognitive/thalamus`).
6. **`services/cortex_scheduler/scheduler.py`** — cortex capacity / any VAD-derived deliberation knob reads the cognitive layer.

### Both nodes
7. **`config/unified_config.yaml`** — new block under `emotional_physics.vad` (§4). Must be identical on both nodes.
8. **`services/mona/app.py`** — rhythm sampler records a second series `cvad_{valence,arousal,dominance}` from `/state.vad_cognitive`; rhythm panel + state widgets render **both** moods so the divergence/convergence is visible (fast jagged affective line vs. smooth gliding cognitive line). Add `mood_cognitive_prompt_text` to `_DASH_CONTENT_FIELDS` so it's stripped from the public mirror like `mood_prompt_text`.
9. **`shared/dashboard_metrics.py`** — register the new `cvad_*` metric keys if the key set is enumerated.

### Tests
10. **`tests/`** — unit test `compare_cognitive` (convergence under a sustained spike; divergence at idle to the two rests), and a regression that Wernicke still sees the fast `vad_global` while Thalamus/Cortex see the smoothed `vad_cognitive`.

---

## 4. Config (proposed defaults — tune in §5 before shipping)

```yaml
emotional_physics:
  vad:
    decay_duration: 10800          # affective, unchanged (3 h)
    baseline: [0.5, 0.5, 0.5]      # affective rest, unchanged
    gravity_weight: 0.7
    cognitive:                     # NEW — slower deliberative layer
      baseline: [0.55, 0.45, 0.60] # DIVERGENT rest: calmer, mildly positive, more composed
      decay_duration: 1800         # 30 min — stabilizes faster toward its own rest
      compare_interval_sec: 12     # frequency of the bidirectional comparison
      pull_from_affective: 0.50    # how strongly cognitive low-passes the current mood
      affective_anchor_pull: 0.08  # gentle reverse pull that damps the per-message jerk
      apply_fatigue_drag: true     # share Layer A's sleep-pressure arousal drop
```

Decision knobs at a glance: **frequency of comparison** = `compare_interval_sec` (12 s); **where each rests** = `baseline` (0.50/0.50/0.50) vs `cognitive.baseline` (0.55/0.45/0.60); **how fast each finds rest** = `decay_duration` 3 h (affective) vs 30 min (cognitive); **how hard they meet in the middle** = `pull_from_affective` (cog←aff, strong) and `affective_anchor_pull` (aff←cog, weak).

---

## 5. Verification before any node edit

Write a throwaway simulation (`scripts/sim_two_layer_mood.py`, not deployed) that drives the two-layer update with: (a) a burst of message spikes then silence, (b) pure idle. Assert: cognitive variance ≪ affective variance under (a); under (b) the two settle to distinct points separated by ≈ `|A_rest − C_rest|`; the affective layer with `affective_anchor_pull` on overshoots less than with it off. Plot both lines — this is the same picture Mona should show. Only after the curves look right do we touch `/opt/syn`.

---

## 6. Architecture documents to update

| Doc | What changes |
|---|---|
| `documents/00-meta/ARCHITECTURE.md` | The per-user-vs-global table (VAD baseline row), Layer 1 section (§245+), the effective-VAD / cross-fade block (§479–494), and the EmoT-grid section (§370+) to describe **two** moods and per-region affective/cognitive sourcing. **Primary.** |
| `DESIGN_DECISIONS.md` | New lettered section recording the two-layer split: divergent rests, comparison frequency, the fixed-point rationale, and why Wernicke stays on Layer A while Thalamus/Cortex move to Layer B. Sits alongside §M (arousal-rest revision), §P (lattice), §S (archival dead-zone). |
| `documents/01-identity-and-self/MODEL_EMOTION_TOPOLOGY.md` | Profiler now resolves an affective *and* a cognitive mood; per-region capacity sourcing. |
| `documents/07-external-interfaces/MONA_AND_PUBLIC_MIRROR_ARCHITECTURE.md` | Rhythm panel/state widgets now show two VAD series; new stripped content field. |
| `documents/03-consciousness-and-cognition/DMN_SCHEDULER_ARCHITECTURE.md` | Note which layer DMN scheduling vs. dream-seed/saturation reads (scheduling → cognitive; pins/saturation → affective). |
| `documents/01-identity-and-self/QUALIA_ARCHITECTURE.md` | If it describes felt mood, note the affective/cognitive distinction. (Confirm on edit.) |
| `CHANGELOG.md`, `WHERE_WE_ARE.md`, `Catch-up.md` | Running status / changelog entry. |

---

## 7. Rollout (when approved)

1. Edit laptop source; run the §5 simulation; iterate config.
2. Land config identically on both nodes (`config/unified_config.yaml`).
3. Push code via `sync_edit.py` to **both** nodes.
4. Restart, per node: Node 2 → `layer1`, `einstein`, `mona`; Node 1 → `thalamus-router`, `cortex`, `dmn`, `mona`.
5. Watch Mona rhythm panel: the cognitive line should glide while the affective line jumps, converging under load and separating at idle.

---

## 8. Open decisions for D

- **Scope of this turn:** deliver this map for review, or proceed straight to implementing across both nodes?
- **Cognitive consumers:** confirm Wernicke stays fully on Layer A (affective) and only Thalamus + Cortex (incl. cortex-region steering) move to Layer B.
- **Cognitive rest point:** confirm the `(0.55, 0.45, 0.60)` "calmer / mildly positive / more composed" anchor, or pick a different divergence.
- **Comparison cadence:** 12 s, or tie it exactly to Mona's 15 s sampler / something else.
