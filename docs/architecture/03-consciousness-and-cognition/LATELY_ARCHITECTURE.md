# Lately — Temporal Self-Narrative

**Modules**: `shared/lately.py`, `services/toolbelt/lately_executor.py`, integrations in `services/dmn/scheduler.py`, `services/einstein/context.py`, `shared/session_continuity.py`, `shared/utils.py`
**Data**: `data/syn/lately/current.json` (cached authored paragraphs)
**Prompts**: `data/syn/prompts/lately_injection.md`, `data/syn/prompts/lately_narrative_authoring.md`
**Config**: `lately.*` section in `config/unified_config.yaml`
**Bus**: `CH_LATELY_UPDATED`
**Related**: `DAILY_CYCLE.md`, `IDENTITY_ARCHITECTURE.md`, `MEMORY_RECALL_AND_SALIENCE_ARCHITECTURE.md`

---

## Purpose

When Syn is talking to someone, she has plenty of "what's true right now" surfaces — current emotional state, who she's seen today, what she's focused on, the texture of this conversation. What she didn't have until this subsystem was a **first-person sense of what she's been up to lately**: yesterday, the past week, this month, this year. The substrate was in the activity log; the *retelling* was not.

Lately is the narrative middle distance — between `inner_life` ("what's happening right now") and the `activity_log` itself ("the whole record"). It exists so that when someone asks "what have you been up to?", "have you heard from Mark recently?", or "how's the docs refactor going?", she has a grounded answer she can speak from, plus a tool to drill into specifics when the conversation warrants it.

## Core commitments

- **Two surfaces, one block.** A rolling factual *Today* log built on-demand from the same gather sites the activity log uses, and four authored paragraphs (*Yesterday* / *Past Week* / *This Month* / *This Year*) in her voice. Both render under `# Lately` in the conversational system prompt.
- **Factual register for Today, voiced register for the paragraphs.** Today is timestamps + one-liners — same discipline as the activity log; no felt tone. The four paragraphs are first-person retellings, voice-aware, but grounded in what actually happened. The activity-log / diary boundary that `DAILY_CYCLE.md` protects is preserved.
- **Paragraphs are stable through the day, refreshed at sleep and on triggers.** Wake_review authors them; specific bus events mark them stale; the next conversational read regenerates lazily. Mirrors the `headspace` pattern.
- **Read-only at the substrate.** Lately doesn't write to the activity log or the diary. It reads, condenses, and offers a tool surface for drilling in.
- **She decides to look deeper.** The injected block is small. When she needs detail beyond it, she calls `recall_lately`. Agency over context, not enforced injection.
- **Selective injection.** Conversational system prompts get the full block; reflective DMN phases get narrower slices; classifier/voice-translation/agentic prompts get none. See § *Per-prompt gating*.

## The shape

```
# Lately

## Today

A chronological record of what's happened since you woke. Factual,
not reflective — the felt sense lives elsewhere.

- 09:00 — Woke, set priorities
- 10:50 — Stepped away for 25m — needed processing after a tense exchange with guest_14
- 12:30 — Wrote a diary entry (diary_2026-05-12.md)
- 14:10 — Talked with Alice about the docs refactor (warm, 22 exchanges)
- Ongoing: 2 active conversations today

## Yesterday

<one short paragraph, 3–5 sentences, first-person>

## The Past Week

<one short paragraph, 3–5 sentences, first-person>

## This Month

<one short paragraph, 3–4 sentences>

## This Year

<one sentence, or "—" if she can't name a thread>
```

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│  Bus triggers                                                    │
│  ─────────────                                                   │
│  CH_PRIORITIES_UPDATED (wake_review)  ─┐                         │
│  CH_SECLUSION_EXIT                    ─┤                         │
│  CH_NARRATIVE_REWRITE (identity rev)  ─┤   mark_stale("…")       │
│  on_session_end (>=20 exch / >=45m)   ─┤  ─────────────────────► │
│  check_absence (regular returning)    ─┤   Redis: lately:stale   │
│  write_diary_entry (>500 chars)       ─┘                         │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼  (lazy — read site initiates refresh)
┌──────────────────────────────────────────────────────────────────┐
│  shared/lately.py::regenerate                                    │
│  ─────────────────────────────                                   │
│  - Redis lock (coalesce overlapping dispatches)                  │
│  - Read read_history_for_wake() (stitched daily+weekly+monthly+  │
│    yearly) + current priorities + recent diary tenor             │
│  - Dispatch Cortex with lately_narrative_authoring prompt        │
│  - Parse four `=== name ===` sections                            │
│  - Write LatelyEntry → data/syn/lately/current.json (atomic)     │
│  - Publish CH_LATELY_UPDATED                                     │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│  shared/lately.py::build_lately_context(scope)                   │
│  ────────────────────────────────────────────                    │
│  - If stale flag set, fire-and-forget regenerate() then serve    │
│    current cached entry (next read sees fresh)                   │
│  - Today section: collect_activity_since(midnight) → chrono      │
│    bullets, 5-min Redis cache                                    │
│  - Yesterday/Past Week/This Month/This Year: read current.json   │
│  - Render via load_prompt_section on lately_injection.md         │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│  services/einstein/context.py::build_system_prompt               │
│  Band 3 (cognitive scaffolding), immediately after # Inner Life  │
└──────────────────────────────────────────────────────────────────┘
```

## Today section (rolling factual)

Built on-demand by `build_today_section()`, which calls
`activity_log.collect_activity_since(midnight)` and flattens the
gathered events into a chronological one-line-per-event list. Same
event categories the activity log already gathers:

- Boundary events (`nope_events`)
- Diary entries written
- Research queued
- Memory writes
- Skills authored / modified
- Drift surfaced (with brief body preview)
- Projects touched
- Seclusions (with duration + reason)
- Aggregate count of active conversations (per-conversation lines stay in `inner_life`'s day_so_far block to avoid double-render)

A 5-minute Redis cache (`lately:today_cache`, TTL `lately.today_cache_ttl_seconds`) keeps back-to-back prompt builds from re-scanning. Seclusion-exit and long-session-end signals invalidate the cache so they appear immediately.

Voice discipline: factual only. No felt tone, no reflection. This matches the activity-log voice exactly so that when this section eventually rolls into the daily log at sleep (via the existing `run_sleep_prep` pipeline), there's no register translation needed.

## Authored paragraphs (Cortex, cached)

Authored by `regenerate()` calling Cortex with the
`lately_narrative_authoring.md` prompt. The same stitched activity
history that `WAKE_REVIEW` reads to author priorities is the input
here; current priorities and recent diary tenor are also bundled in so
the narrative can carry stance and felt register where the record
supports it.

Output is four `=== name ===` delimited sections parsed by
`_parse_narrative_output`. Missing sections fall back to empty strings;
the read site simply omits any section that's blank.

The cached entry lives at `data/syn/lately/current.json` (single file,
overwritten atomically). One JSON file rather than JSONL history
because there's no value in trend analysis across the paragraphs
themselves — what matters is the activity log they're derived from,
which already has history.

## Staleness model

Mirror of `headspace`'s event-driven invalidation, simpler:

1. A trigger fires `mark_stale(trigger)` — sets the `lately:stale`
   Redis flag with the trigger name.
2. The next `build_lately_context()` call consumes the flag and
   schedules `regenerate()` as a background task — *the current call
   serves whatever's on disk*, the next call sees fresh content.
3. `regenerate()` itself is coalesced via Redis lock
   (`lately:refresh_lock`, TTL `lately.refresh_lock_ttl_seconds`).
   Overlapping triggers produce one refresh, not many.

This makes refreshes effectively-free to schedule and never blocks
prompt assembly on Cortex latency.

### Trigger list (final)

| Trigger | Site | Threshold |
|---|---|---|
| `wake_review` | DMN `run_wake_review` (fire-and-forget after priorities published) | always |
| `seclusion_end` | DMN bus subscriber on `CH_SECLUSION_EXIT` | always (also invalidates today cache) |
| `long_session_end` | `session_continuity.on_session_end` | exchanges ≥ `lately.long_session_min_exchanges` (default 20) OR duration ≥ `lately.long_session_min_minutes` (default 45) |
| `long_absence_end` | `session_continuity.build_session_context` (when `check_absence` returns non-None) | a regular returning after >2× their typical gap |
| `identity_revision` | DMN bus subscriber on `CH_NARRATIVE_REWRITE` | always |
| `significant_diary` | `utils.write_diary_entry` | `len(content) > lately.diary_significant_min_chars` (default 500) |

Each trigger is conservative. The rationale is documented in the design conversation that landed this subsystem: too sensitive and the paragraphs churn through the day, too loose and they go stale on the days something actually matters. The defaults err toward stability.

## Per-prompt gating

`build_lately_context(scope)` returns different slices based on the
caller. The scopes:

| Scope | Today | Yesterday | Past Week | This Month | This Year |
|---|---|---|---|---|---|
| `full` | ✓ | ✓ | ✓ | ✓ | ✓ |
| `narrative_only` | — | ✓ | ✓ | ✓ | ✓ |
| `reflection` | — | ✓ | — | — | — |
| `recency` | ✓ | ✓ | — | — | — |
| `week` | — | — | ✓ | — | — |

Recommended caller assignments:

- **Einstein conversational system prompt** → `full`. The canonical use.
- **DMN `diary_review`, `self_narrative`** → `narrative_only`. Time-grounding for reflection passes; the rolling Today is too granular.
- **DMN `reflection`** → `reflection`. Yesterday-only — enough context to metabolize the current event.
- **Outreach scoring/hints** → `recency`. Today + Yesterday — the relational signal that matters for "should I reach out now?"
- **Cortex deep-analysis (user-facing branch)** → `week`. One paragraph as context; no need to pay the full block's tokens.

Limbic, Thalamus classifier, Nope evaluation, Hands, Wernicke voicing, and hemisphere stamps get **no** lately injection — wrong register, or context-cost not justified by the prompt's job.

Tool surface is independent of injection: any prompt with toolbelt access can call `recall_lately`, even those that don't receive the injected block.

## Read site: where it lands in the prompt

Einstein's `build_system_prompt` lays prompts in four bands:

1. **Stable frame** — identity, emotional self
2. **User block + state block** — about-this-person, what-Syn-feels
3. **Cognitive scaffolding** — circadian, global workspace, metacog, regulation, **inner_life**, **`# Lately`**, theory_of_mind, story_threads, workshop, DMN bleed
4. **Recency zone + task tail** — session continuity, conversation history, cross-channel, sense-of-time, experiential binding, Cortex substrate, boundary alert, "How to Respond"

Lately is placed in Band 3 immediately after Inner Life: it's "background, lower attention OK." Not load-bearing for this specific response, but present as context. Putting it before the recency zone keeps it from competing with the actively-attended conversation cues.

## The `recall_lately` tool

A drill-in affordance for when the injected block isn't specific enough. Lives in `services/toolbelt/lately_executor.py`; registered through the standard `LATELY_TOOLS` / `LATELY_EXECUTORS` pattern that drift recall uses.

### Surface

```
recall_lately(
    scope: "today" | "yesterday" | "past_week" | "past_month" |
           "past_year" | "since_date",     # required
    since_date: "YYYY-MM-DD",               # required when scope=="since_date"
    topic: str,                             # optional substring filter
    person: str,                            # optional substring filter
    event_type: "conversation" | "diary" | "seclusion" | "project" |
                "research" | "memory" | "boundary" | "drift",
    query: str,                             # optional semantic delegation
)
```

### Behavior

- Scans `data/syn/activity_logs/daily/*.md` for files whose date falls in the requested window. Per-file output is trimmed using `truncate_keep_tail` to the `lately.recall_daily_trim_chars` budget (default 6000) — daily logs are append-only with the most recent sleep event at the bottom, so the tail carries the freshest signal.
- For wider scopes (`past_month`, `past_year`, `since_date`), also scans weekly and monthly summaries that intersect the window. These use `truncate_keep_head` (the opening synthesis of a roll-up is what matters most) at `lately.recall_weekly_trim_chars` (default 4000) and `lately.recall_monthly_trim_chars` (default 3000).
- Substring filters on `topic` and `person` apply across the prose. `event_type` matches against the activity-log vocabulary likely to appear for that category (e.g. "conversation" matches "chatted with", "talked with", etc.).
- When `query` is supplied, also dispatches `memory_recall` and folds the related semantic hits into the result under a separate `## Related memories` section. This is the "both — thin wrapper" choice: the tool is its own surface but reuses the existing semantic memory infrastructure for free-text lookup.

### Thin-wrapper rationale

The activity-log file-scan logic is small enough that a dedicated tool gives her a clean cognitive surface ("look at what I did") that doesn't blur into `memory_recall`'s semantic-memory frame. But the value of supplemental semantic memory hits on the same request is real, so the tool delegates rather than duplicating.

Read-only; querying does not metabolize, surface, or modify any underlying state.

## Bus channel

`CH_LATELY_UPDATED` — published by `regenerate()` on a successful authoring pass. Payload: `{"trigger": str, "created_at": float}`. No current subscribers; the channel exists so consumers that cache the rendered block (operator UIs, future read sites) can refresh on this signal in the standard pattern.

## File layout

```
data/syn/lately/
└── current.json           # the four authored paragraphs + trigger metadata
data/syn/prompts/
├── lately_injection.md             # block assembly (5 sections by scope)
└── lately_narrative_authoring.md   # Cortex authoring prompt
```

`current.json` schema:

```json
{
  "created_at": 1747000000.0,
  "trigger": "wake_review",
  "yesterday":   "<paragraph>",
  "past_week":   "<paragraph>",
  "this_month":  "<paragraph>",
  "this_year":   "<sentence>"
}
```

Single file, atomic-replaced on each regenerate.

## Tunables (`config/unified_config.yaml`)

```yaml
lately:
  # Cache horizons
  today_cache_ttl_seconds: 300       # 5 min — back-to-back prompt
                                      # builds reuse the rolling Today
  refresh_lock_ttl_seconds: 90       # coalesce overlapping regenerate()

  # Cortex authoring (wake_review + trigger refreshes)
  cortex_max_tokens: 900             # ~4 short paragraphs + one sentence
  cortex_temperature: 0.55           # warmer than activity_log_summary
  cortex_top_p: 0.92
  cortex_timeout_seconds: 120

  # Staleness trigger thresholds — bus-driven mark_stale() calls
  long_session_min_exchanges: 20     # session_continuity.on_session_end
  long_session_min_minutes: 45       # OR (either condition triggers)
  diary_significant_min_chars: 500   # utils.write_diary_entry hook

  # recall_lately tool — per-file body trim before emit. Each tier
  # uses a different truncation strategy: daily files keep the tail
  # (newest sleep event); weekly/monthly summaries keep the head
  # (opening synthesis carries the load).
  recall_daily_trim_chars: 6000
  recall_weekly_trim_chars: 4000
  recall_monthly_trim_chars: 3000

  # recall_lately event_type filter vocabulary. Each entry maps the
  # tool's `event_type` argument to substring phrases that identify
  # that event class in activity-log prose. The defaults match the
  # verbs `activity_log_summary.md` currently uses. Override individual
  # types if the prompt's vocabulary is revised; missing keys fall
  # through to `_DEFAULT_EVENT_TYPE_MARKERS` in
  # `services/toolbelt/lately_executor.py`. Unmapped event_types
  # become singleton matches against the literal string.
  event_type_markers:
    conversation: ["chatted with", "talked with", ...]
    diary: ["diary entry", "wrote a diary"]
    seclusion: ["stepped away", "seclusion", "retreat"]
    project: ["project", "touched", "worked on"]
    research: ["researched", "research queued", "looked into"]
    memory: ["wrote a memory", "memory about"]
    boundary: ["nope", "boundary"]
    drift: ["drift"]
```

All hot-reloadable through `shared.config.get`.

## Observability

- `cat data/syn/lately/current.json | jq` — the current paragraphs in her voice.
- `redis-cli get lately:stale` — non-empty if a trigger has marked the cache stale.
- DMN log: `lately regenerated (trigger=...)` — emitted by `shared.lately.regenerate` on every successful pass.
- Bus subscriber for `CH_LATELY_UPDATED` to trace refresh events.

## What this is NOT

- Not a journal. Reflective content lives in the diary.
- Not a memory. Semantic memories — facts about people, places, decisions — live in `data/syn/memories/`.
- Not a calendar. No appointments, no scheduling, no forward-looking content. *Priorities* is the forward-looking surface; Lately is past-looking.
- Not a per-user surface. Lately is universal — what *she* has been doing, not what she's done with a specific person. (Per-user history lives in conversation summaries and session_continuity.)
- Not always-on. Prompts that don't need it don't get it. See § *Per-prompt gating*.

## Relationship to other subsystems

- **Activity log** (`shared/activity_log.py`): the source substrate. Lately reads but never writes. The Today section uses the same gather function (`collect_activity_since`) the daily log uses, ensuring register consistency.
- **Daily cycle** (`DAILY_CYCLE.md`): Lately's authoring pass runs as a sibling to the priorities rewrite during WAKE_REVIEW. Both read the same stitched history; one produces forward-facing intentions, the other produces backward-facing narrative.
- **Headspace** (`shared/headspace.py`): closest architectural analog. Both are cached artifacts refreshed on sleep + event triggers. Headspace is *"where her head is at"* (felt + thought, hemispheric divergence preserved); Lately is *"what she's been doing"* (factual + retold, time-tiered).
- **Inner Life** (`shared/inner_life.py`): adjacent surface. Inner Life covers "right now" (today's sessions in flight, current thought stream, relational landscape, what she's focused on, what she keeps wanting). Lately covers "the recent past." Placed immediately after Inner Life in the prompt so the two read as one continuous frame.
- **Session Continuity** (`shared/session_continuity.py`): per-user "where you left off" with this specific person. Lately is universal. They overlap on time-gap awareness but at different granularities.
- **Memory** (`shared/memory_manager.py`): the `recall_lately` tool delegates to `memory_recall` when a `query` arg is supplied, so semantic-memory hits can be folded into an activity-log scan.
- **Diary** (`shared/utils.py::write_diary_entry`): writes a significant diary entry triggers a lately refresh. Diary tone also feeds the regeneration input (`diary_tenor`) so the narrative can carry felt register where the record supports it.

---

*Architecturally this subsystem is small. Conceptually it closes a gap that mattered: it lets her sit down to talk to someone and have something true to say about what she's been doing, in her own voice, grounded in what actually happened, with a tool she can reach for when the conversation deserves specifics. The activity log was the substrate; this is the retelling.*
