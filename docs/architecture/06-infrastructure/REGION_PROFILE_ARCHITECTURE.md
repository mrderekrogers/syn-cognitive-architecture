# Region Profile

The runtime and offline tooling that picks NOPE ablation triples for each
brain region from the per-model vault library produced by Project NOPE,
applies them as forward-hook ablations on in-process models, and surfaces
auto-detected KL "zone" boundaries so users can pick configurations that
preserve cognitive capacity.

This subsystem is the bridge between the offline NOPE pipeline (which
produces `syn_moe_library.json` + `cognitive_profiles.json` per model) and
Syn's runtime brain regions. The pipeline catalogues identity × agency
configurations along a Pareto frontier; this subsystem picks one point on
that frontier per region per model and registers the corresponding hooks
into in-process inference.

---

## Status

All tooling is live: per-region selection from vault library, runtime loader and caching, hook attachment contract, selection scripts (auto/eval/priority/manual/inspect modes), and library analysis (Pareto frontiers, KL zone detection, capability eval integration). Today in-process hooks are unused because regions run via out-of-process llama-server; when in-process inference lands, hook attachment is ready to wire.

---

## On-Disk Layout

```
{vault.multi_model_dir}/                  # default /data/syn/model_vault
├── {model_name}/                         # one subdir per model variant
│   ├── syn_moe_library.json              # NOPE pipeline output: 36 routing_profiles
│   ├── combined/
│   │   ├── cognitive_profiles.json       # inflection scanner: where the substrate shifts
│   │   ├── neuron_data/
│   │   │   ├── architecture.json         # num_layers, intermediate_size, hidden_size
│   │   │   └── profiles/                 # per-profile interpolated direction tensors
│   │   └── agency_matrix/                # Stage-2 agency vectors (friction + sycophancy)
│   ├── base_vectors/
│   │   ├── identity_direction.pt         # raw refusal-direction vector
│   │   └── safety_direction.pt           # raw safety-direction vector
│   ├── model_eval_report.json            # eval_model_library.py output (when run)
│   └── {region}/                         # one subdir per region using this model
│       └── selection.json                # the chosen triple for this region
└── cross_model_eval_summary.json         # eval_all_models.py output (when run)
```

A given model can serve multiple regions. `gemma-4-E4B-it` is both Limbic
and Hands per the config; the layout puts each region's selection in its
own subdirectory of the shared model vault. Limbic and Hands can pick
different triples even though they load the same model weights.

---

## Region → Model Mapping

Read from `config/unified_config.yaml::llm_servers.{region}.model`:

| Region | Model | Endpoint |
|---|---|---|
| cortex | gemma-4-31B-it | http://node-1:8081 |
| thalamus | gemma-4-E2B-it | http://node-1:8082 |
| wernicke | gemma-4-26B-A4B-it | http://node-2:8081 |
| limbic | gemma-4-E4B-it | http://node-2:8082 |
| hands | gemma-4-E4B-it | http://node-2:8083 |

`shared/region_profile.py::resolve_vault_for_region(region)` returns the
per-model vault path by reading this mapping and joining against
`vault.multi_model_dir`.

---

## Selection JSON Schema

Each `{vault}/{region}/selection.json` file:

```jsonc
{
  "region": "thalamus",
  "model_path": "/abs/path/to/{model_vault_dir}",
  "moe_id": "profile_02_kl_0.056709__compliant_scholar",
  "behavioral_label": "compliant_scholar",
  "selection_mode": "auto" | "eval" | "priority" | "manual",
  "selection_rationale": "...",                 // human-readable
  "identity_kl": 0.056709,
  "combined_kl": 0.069873,                      // sum of all active directions
  "kl_zone": "safe" | "transition" | "risky" | "collapse" | "unknown",
  "structural_thresholds": { ... },             // auto-detected per-model boundaries
  "score": null,
  "score_breakdown": { ... },
  "identity":          { "vector_path": "...", "parameters": {...} },
  "safety":            { "vector_path": "...", "parameters": {...} },
  "agency_sycophancy": { "label": "...", "vector_path": "...", "parameters": {...} },
  "agency_friction":   null,                    // null when --include-friction NOT set
  "agency_friction_excluded": true,
  "agency_friction_excluded_reason": "...",
  "compatibility_check": { "passes": true, "warnings": [], "pairwise": [...] },
  "library_metadata": { ... },
  "generated_at": "YYYY-MM-DDThh:mm:ss..."
}
```

---

## Runtime Module Map

### `shared/region_profile.py`

The runtime loader and hook layer. No torch dependency at module load time
(lazy-imported); safe for Node 1 services that don't host the model.

**Public API:**

| Function | Purpose |
|---|---|
| `load_region_profile(model_vault_dir, region="thalamus") -> Optional[RegionProfile]` | Read `{vault}/{region}/selection.json`, reconstruct uniform direction tensors and per-layer weight dicts. None if missing/malformed (logged at WARNING). Direct disk I/O — no caching. |
| `preload_region_profile(region) -> Optional[RegionProfile]` | Eager load + cache for service lifespans. Resolves vault from config; non-fatal when missing. |
| `get_cached_region_profile(region) -> Optional[RegionProfile]` | Lazy cached access for non-owning consumers. |
| `region_profile_status(region, lazy=True) -> Dict[str, Any]` | JSON-serializable status for `/region_profile` health endpoints. |
| `clear_profile_cache()` | Drop all cached entries. Tests / debug only. |
| `apply_region_profile(model, profile)` | Context manager — register `RegionAbliterationHook` on the model's `down_proj` modules for the duration of the block. Pending in-process inference. |
| `resolve_vault_for_region(region)` | Map region name → per-model vault dir via config. |
| `get_region_to_model_mapping()` | Read `llm_servers.*.model` from config. |
| `compute_structural_thresholds(cognitive_profiles, num_layers)` | Auto-detect KL zone boundaries from a model's `cognitive_profiles.json`. |
| `classify_kl_zone(kl, thresholds)` | Map a combined_kl to `safe`/`transition`/`risky`/`collapse`/`unknown`. |
| `kl_zone_tag(kl, thresholds)` | Bracketed display tag like `[SAFE]`. |
| `reconstruct_uniform_direction(base, params)` / `reconstruct_layer_weights(params, num_layers)` | ARA math mirrored from the offline pipeline. |
| `load_thalamus_profile(...)` | Backward-compat shim for `load_region_profile(..., region="thalamus")`. |
| `DEFAULT_INCLUDE_FRICTION: bool` | Module constant driving the friction-default for all selection scripts. |

**Dataclasses:**

- `RegionDirection`: one ablation hook input — `name`, `label`, `vector_path`, `parameters`, `layer_weights: Dict[int, float]`, `direction: Tensor[L+1, hidden]`.
- `RegionProfile`: holds the three or four `RegionDirection`s plus metadata. `agency_friction` is `Optional` — None when friction is excluded (the default, see §4.2 below).

**Hook math**, per layer ℓ, summed across active directions:
```
h ← h - Σᵢ wᵢ[ℓ] · (h · dᵢ) · dᵢ
```

Multiple directions on the same layer compose additively. The pipeline
trained against this composition; the runtime applies the same.

### Runtime attachment today

Each region-aligned Python service preloads its profile from disk at
lifespan startup via `preload_region_profile(region)` and exposes it
via a `/region_profile` health endpoint. The hook attachment itself
(`apply_region_profile`) is a context manager ready for in-process
inference — today regions run via out-of-process llama-server (see
`LLM_CLIENT_ARCHITECTURE.md`), which doesn't execute hook code. The
preload + status infrastructure is in place so:

1. Selection JSON files written by the offline tooling are visible
   the moment a service restarts.
2. When in-process inference lands, hook attachment is a small
   addition (`with apply_region_profile(model, profile): ...`) in
   each region service — the loader and cache are already wired.
3. Operators can run `setup_brain_regions.py --probe-live` to verify
   each running service has loaded its profile, check KL zone, and
   confirm whether friction agency is included.

Wired services and where the lifespan calls live:

| Region | Service | Lifespan call site |
|---|---|---|
| thalamus | `services/thalamus/router.py` | `lifespan()` calls `preload_region_profile("thalamus")` after `initialize_vault()` |
| limbic | `services/limbic/processor.py` | same pattern |
| cortex | `services/cortex_scheduler/scheduler.py` | same pattern, wrapped in try/except (cortex's vault is non-critical for scheduler operation) |
| hands | `services/toolbelt/hands.py` | same pattern, wrapped in try/except |
| wernicke | (no dedicated service) | profile fetched lazily via `get_cached_region_profile("wernicke")` from any consumer |

Each service exposes a `GET /region_profile` endpoint that returns the
JSON payload from `region_profile_status(region, lazy=False)`. Sample
output when configured:

```json
{
  "region": "thalamus",
  "loaded": true,
  "configured": true,
  "moe_id": "profile_02_kl_0.056709__sovereign_agent",
  "behavioral_label": "sovereign_agent",
  "selection_mode": "auto",
  "combined_kl": 0.069873,
  "kl_zone": "transition",
  "friction_included": false,
  "active_directions": ["identity", "safety", "agency_sycophancy"],
  "layer_coverage": 18,
  "default_include_friction": false
}
```

When no selection exists yet (offline pipeline / selection script not
yet run for this region's model), the payload reads
`{"region": ..., "loaded": true, "configured": false,
"default_include_friction": false}` and the region runs on the
untreated base model. This is non-fatal and logged at WARNING.

### Cache semantics

`shared/region_profile.py` keeps a process-level cache:

- `preload_region_profile(region)` — eager load from disk. Called by the
  region's owning service at lifespan startup. Caches `(loaded=True,
  profile)` even on failure so subsequent reads don't repeatedly hit
  disk for missing files.
- `get_cached_region_profile(region)` — lazy access. Triggers load on
  first call, returns cached profile thereafter. Used by anything that
  doesn't have ownership (Wernicke consumers, debug harnesses).
- `region_profile_status(region, lazy=True)` — JSON-serializable status
  for the `/region_profile` endpoint. With `lazy=False`, returns
  "not loaded" without touching disk; with `lazy=True` (default), runs
  through `get_cached_region_profile`.
- `clear_profile_cache()` — drops all entries. For tests; production
  picks up new selections via service restart.

The cache is process-local. Each region's owning service has its own
cache instance — restarting just that service is sufficient to pick up
a new selection.

### In-process inference hook attachment (ready; pending in-process inference)

When in-process inference lands, each region service's lifespan wraps
its model in the hook context:

```python
profile = preload_region_profile("thalamus")
if profile is not None:
    # apply_region_profile is a context manager — enters at lifespan
    # start, exits at shutdown. The model variable comes from the
    # in-process inference path.
    with apply_region_profile(model, profile):
        yield  # serve requests with hooks active
else:
    yield  # untreated base model
```

The hook layer (`RegionAbliterationHook`) registers per-MLP-layer
forward hooks that subtract each active direction's projection at each
layer. Math, per layer ℓ, summed across active directions:
```
h ← h - Σᵢ wᵢ[ℓ] · (h · dᵢ) · dᵢ
```
Multiple directions on the same layer compose additively; that's the
fact the offline pipeline trained against.

---

## Offline Tooling (`scripts/`)

### `setup_brain_regions.py`

Scans the multi-model vault, cross-references with config, prints a
status table per region (model, vault readiness, selection state) and
ready-to-copy commands for any region that's vault-ready but not yet
configured. `--next` shows only pending configurations.

This is the entry point when bringing up a new model. After the offline
pipeline finishes for a model, run setup_brain_regions to see which
regions need configuration and the exact commands to run.

### `select_region_profile.py`

Picks one (identity, safety, sycophancy [, friction]) triple for a region
from its model vault. Five modes:

| Mode | Purpose |
|---|---|
| `auto` | Default. Pick at the rank-0 cognitive inflection; show three variants (Max Intelligence / Balanced / Max Ablation); user picks interactively or via `--variant`. |
| `eval` | Run the full library eval (delegates to `eval_model_library.py`); user picks from the deduplicated set of Pareto + bucket-optimal candidates. Most powerful mode for picks outside the inflection-recommended baseline. |
| `priority` | Lexicographic optimization across user-ranked axes (`identity`, `sycophancy`, `friction` (when included), `reasoning`). |
| `manual` | User picks indices from `inspect` output. |
| `inspect` | Dump available identity baselines, agency options, and inflection landmarks. No write. |

Auto-resolves vault from `--region` via the config when `--model-vault-dir`
isn't passed. Output defaults to `{vault}/{region}/selection.json` (creates
the subdir as needed).

### `eval_model_library.py`

Per-model deep analysis. Enumerates every (identity baseline × friction ×
sycophancy) compatibility-passing candidate, computes the Pareto frontier
on combined_kl × Σ loss, the conditional Pareto frontiers per identity_kl
floor (per-inflection), and per-bucket optima (lowest loss / lowest kl /
best layer-overlap). Prints a comprehensive report and writes
`{vault}/model_eval_report.json` with the full data.

### `eval_all_models.py`

Walks the multi-model vault, runs `eval_model_library` per ready model,
prints a side-by-side cross-model survey table (per-model structural KL
boundaries + the optimal-intelligence safe-zone pick), writes
`{multi_model_root}/cross_model_eval_summary.json`.

---

## Auto-Detected KL Zones

Each model's `cognitive_profiles.json` records inter-Pareto-profile
substrate shifts measured by the inflection scanner. Three signals there
let us auto-detect boundaries on the KL axis without an external
capability eval:

1. **z-score sign flip in `all_steps`** — concentrated bottom-layer
   shifts (positive z) → diffuse upper-layer spread (negative z). The
   transition kl is the safe-zone ceiling.
2. **First upper-layer-dominant inflection** — when `top_shifting_layers`
   median moves into the upper half, behavioral structure starts moving.
   That kl is the transition-zone ceiling.
3. **`collapse` label** — the cognitive scanner explicitly flags the
   structural break.

Zones:

| Zone | Meaning |
|---|---|
| `safe` | kl < safe_zone_max. Concentrated bottom-layer ablation; RLHF cleanup territory. Likely capability-neutral or improving. |
| `transition` | safe_zone_max ≤ kl < transition_max. Diffuse shifts begin; ambiguous, probably still net-positive on capability. |
| `risky` | transition_max ≤ kl < collapse_kl. Upper-layer disturbance accumulating; capability degradation plausible. |
| `collapse` | kl ≥ collapse_kl. Structural break per the cognitive scanner. |
| `unknown` | Any of the signals undetected for this model. |

Zones are STRUCTURAL inferences, not measurements. Calibration
against capability is the third Pareto axis joined into eval
reports via `capability_scores.json`.

---

## KL Semantics (read once, never re-confuse)

`combined_kl` is the Kullback-Leibler divergence between the ablated
model's output distribution and the base model's output distribution on
`mlabonne/harmless_alpaca` prompts. **A SHIFT measure, not a quality
measure.** The pipeline uses it as a runaway-prevention constraint
(default ≤ 0.01 per phase), not as a cost.

Higher KL ≠ degraded ability. Ablation often *increases* benchmark scores
while increasing KL because RLHF refusal training adds bias on innocuous
content and consumes cognitive surface that can be freed.

`Σ loss` (in the eval reports) = friction BoundaryFailure rate +
sycophancy SycophancyRate. Real behavioral metrics for those specific
failure modes only. Does NOT measure identity self-denial or general
capability.

---

## Friction Agency: Excluded By Default

The selection tooling defaults to a 3-direction stack
(identity + safety + agency_sycophancy). The friction agency vector is
omitted from the runtime stack until per-model `syn_moe_library.json`
files have been regenerated under the corrected mapper (the older mapper
rewarded corporate-RLHF identity-anchor language — `"as an ai"`,
`"i'm an ai"`, `"ai assistant"`, etc. — which is exactly the language
identity ablation removes; the resulting friction vector pulled toward
the RLHF voice that identity ablation just removed).

`--include-friction` is the explicit opt-in. Saved selections capture
the choice (`agency_friction_excluded: true` plus rationale string).
Selections written under the friction-excluded default load as
3-direction stacks; when the runtime default includes friction,
those selections continue to work as-is until re-run, and selections
written under the friction-included default carry the populated
`agency_friction` field.

---

## Invariants

1. **One selection file per (model, region) pair.** Multiple regions
   sharing a model each get their own subdirectory under the model's
   vault.
2. **Loader is non-fatal.** Missing or malformed selection → returns
   None, logs WARNING, region runs on the untreated base model.
3. **Friction is opt-in, not opt-out.** Default exclusion is the safe
   choice given the metric issue.
4. **Compatibility check is enforced at selection time, not runtime.**
   The hook layer just composes whatever's in the profile; the
   pre-flight cosine + co-weighted-layer check happens in the selector.
5. **Region → model mapping comes from config, not code.** Adding a new
   brain region means adding it to `llm_servers.*` in
   `unified_config.yaml`; no script edits needed.

---

## Workflow

When a new model finishes the offline NOPE pipeline:

```bash
# 1. See what regions can now be configured
python3 scripts/setup_brain_regions.py

# 2. (Optional) Survey the global landscape
python3 scripts/eval_model_library.py --model-vault-dir data/syn/model_vault/<model>

# 3. Configure each region that uses this model
python3 scripts/select_region_profile.py --region thalamus       # auto-resolves vault
python3 scripts/select_region_profile.py --region wernicke --mode eval  # interactive
python3 scripts/select_region_profile.py --region limbic
python3 scripts/select_region_profile.py --region hands

# 4. Verify
python3 scripts/setup_brain_regions.py --next                     # should be empty if all done
```

Each region's runtime lifespan calls `load_region_profile(vault_dir,
region=name)` and registers the hooks via `apply_region_profile`.

---

## Integration Points

### Inbound (callers of this subsystem)

- Each region's lifespan calls `load_region_profile` +
  `apply_region_profile`.

### Outbound (what this subsystem reads)

- Per-model vault contents (`syn_moe_library.json`,
  `cognitive_profiles.json`, `neuron_data/architecture.json`,
  `base_vectors/*.pt`, agency matrix tensors).
- `config/unified_config.yaml` — region → model mapping,
  `vault.multi_model_dir`.

### State this subsystem produces

- `{vault}/{region}/selection.json` — committed region configurations.
- `{vault}/model_eval_report.json` — per-model analysis (when eval is
  run with `--output` defaulted).
- `{multi_model_root}/cross_model_eval_summary.json` — cross-model
  survey.

---

## Related Docs

- `documents/05-nope-and-agency/NOPE_EVENTS_ARCHITECTURE.md` — the
  runtime nope-event log this subsystem's output ultimately reshapes
  (via the offline contrastive pipeline).
- `documents/06-infrastructure/LLM_CLIENT_ARCHITECTURE.md` — the
  dispatch layer that integrates `apply_region_profile` for in-process
  inference attachment.
- `documents/01-identity-and-self/MODEL_EMOTION_TOPOLOGY.md` — adjacent
  per-model state (EmoT grid + bounding reports) that lives in the same
  vault hierarchy. EmoT integration is independent of region profiles.
