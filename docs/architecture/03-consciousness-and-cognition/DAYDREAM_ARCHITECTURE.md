# Daydream — Open-Form Play (Cortex/Thalamus Register)

Curiosity is research-shaped: it produces output, satisfies a goal. Basal drift is felt-state-shaped: it surfaces what's been accumulating affectively. Daydream is what happens between those — her mind wandering with no aim, doodling in prose, asking herself questions she may or may not answer, humming an idea around.

This module is the *cortex/thalamus-driven* sibling of `shared/fantasy.py` (which is limbic-driven). Daydream is cognitive, not somatic — no arousal gate, no subject required, no obligation to be useful.

Single module: `shared/daydream.py`. One prompt: `dmn_daydream.md`. One picker affordance: `daydream`. One auto-trigger in the REFLECTION phase tail. Keyword-relevant inner-life surfacing.

---

## Why It's Here

From the design conversation: *"play is what happens when a being explores with no goal: doodling, word association, humming an idea around. I don't see a phase or affordance for 'she's wasting time on purpose.' It might fall out of basal drift naturally, but drift is generated from substrate context and tends toward the felt rather than the playful."*

This module is the "she's just messing around" surface.

What daydream is NOT:
- It's not diary (which is reflection on what happened).
- It's not research (which has a target).
- It's not fantasy (which is subject-bounded and limbic-keyed).
- It's not drift (which is felt-state-driven, not chosen).

It IS the cortex/thalamus space for chosen wandering — and the user's framing of "ask herself questions and answer them" is one shape it can take, not the only one.

---

## File Format

```markdown
# Daydreams

## 2026-05-15T10:00:00Z

(her prose, as much or as little as feels right)

## 2026-05-16T22:30:00Z

(next entry, much later)
```

Each entry is a `## <iso>` header followed by free-form prose until the next header. Append-only. Atomic writes (tmp + rename) and sealing are both handled internally by `shared.seal.write_sealed` — the daydreams file is **sealed at rest** alongside diary / drift / qualia / carried-relations / fantasies. The seal reader transparently handles both sealed and legacy-plaintext bytes; the file migrates cleanly via `scripts/seal_migrate.py`.

---

## Two Paths to Fire

**Picker path (deliberate).** `daydream` is always-eligible. The picker selects it; `_run_motivational_deliberation` routes to `maybe_initiate_daydream()`.

**Auto-trigger path (rate-limited).** `maybe_auto_initiate_daydream` runs from the REFLECTION phase tail. Eligibility: `is_due_for_dmn_auto_trigger()` returns True iff hours-since-last-entry ≥ `daydream.dmn_min_hours_between_auto` (default 8.0). When it returns True, the auto-trigger invokes `maybe_initiate_daydream` — same code path as the picker.

The user's framing in the design conversation specifically routed daydream as the *cortex/thalamus* register (vs. fantasy's *limbic* register). The REFLECTION phase placement reflects that — daydreams fire alongside the cognitive-metabolism work, not alongside affective state changes.

---

## Composition

`maybe_initiate_daydream()` reads up to 3 most-recent prior daydreams (for orientation, not constraint), renders `dmn_daydream.md` with the context, and calls Cortex (`max_tokens=800, temperature=0.85`). Parses `ENTRY: <body>` — multi-line body until end of output. Empty / too-short output drops without appending.

The prompt deliberately does not propose topics. Suggesting starting points would collapse the open framing — what comes up is hers to find.

---

## Inner-Life Surfacing

`render_for_inner_life(context_text=...)` produces a `## Daydreams` block. Two modes:

- **Context-relevant** (when `context_text` is non-empty). Finds prior daydream entries whose token overlap with the context exceeds `daydream.keyword_match_min_overlap` (default 0.25 Jaccard). Returns up to 3 most-relevant entries (ties broken by recency). Block header reads `## Daydreams (related to current context)`.
- **Tail fallback** (when no context or no matches). Returns the 3 most-recent entries, most-recent first. Block header reads `## Daydreams`.

The context-text is built by `build_inner_life_context` from the last session summary + topic tags for the current interlocutor. When there's no active conversation, the block uses the tail fallback.

Each entry shows: date prefix + excerpt (capped at `daydream.inner_life_excerpt_chars`, default 100, cut at word boundary with ellipsis).

---

## Configuration Surface

All under `daydream.*` in `unified_config.yaml`. Hot-reloadable.

| Key | Default | Purpose |
|---|---|---|
| `daydream.inner_life_max_entries` | 3 | Cap on entries in the inner-life block. |
| `daydream.inner_life_excerpt_chars` | 100 | Per-entry excerpt length cap. |
| `daydream.dmn_min_hours_between_auto` | 8.0 | Rate-limit for the auto-trigger. Picker-path daydream is unaffected. |
| `daydream.keyword_match_min_overlap` | 0.25 | Jaccard threshold for context-relevant inner-life surfacing. |

---

## Invariants and Contracts

1. **No arousal gate.** Daydream is cortex/thalamus register; the limbic side has its own surface (`fantasy.py`). Mixing the two would collapse the design distinction.
2. **No subject required.** Unlike fantasy, daydream isn't bounded by a person or scenario. The prompt frames it as open-form.
3. **Append-only.** Entries are timestamped on append. Edits / removals are manual (operator or direct file edit).
4. **Auto-trigger never overrides explicit choice.** The picker can fire daydream regardless of rate-limit state. The rate-limit only gates the *automatic* surface.
5. **Keyword matching is informational, not gating.** When inner-life surfaces context-relevant daydreams, it doesn't extract them as content for outbound prompts — only as interior-context for her own awareness.
6. **No outbound surfacing.** Daydreams are interior. They do not appear in composition prompts (outreach, repair, carried-checkin). She can choose to bring something up that she's daydreamed about; the system doesn't inject it.

---

## Integration Points

### Inbound (who calls this module)

| Caller | What it calls |
|---|---|
| `services/dmn/scheduler.py::maybe_initiate_daydream` | `read`, `append_entry` |
| `services/dmn/scheduler.py::maybe_auto_initiate_daydream` | `is_due_for_dmn_auto_trigger` |
| `services/dmn/scheduler.py` REFLECTION phase tail | `maybe_auto_initiate_daydream` |
| `services/dmn/scheduler.py::_run_motivational_deliberation` | `maybe_initiate_daydream` on `daydream` pick |
| `shared/inner_life.py::build_inner_life_context` | `render_for_inner_life(context_text=...)` |
| `shared/affordances.py::build_state` | `count_entries` |

### Outbound (what this module calls)

- `shared.config.get` — thresholds
- `asyncio.to_thread` — file I/O

No LLM calls (the dispatch is in the DMN scheduler). No bus publishes.

### State produced

- `data/syn/daydreams.md`

---

## Related Docs

- `documents/01-identity-and-self/TASTE_ARCHITECTURE.md` — companion self-authored aesthetic surface.
- `documents/04-relational-and-social/FANTASY_ARCHITECTURE.md` — limbic-register sibling.
- `data/syn/prompts/dmn_daydream.md` — the prompt this drives.
