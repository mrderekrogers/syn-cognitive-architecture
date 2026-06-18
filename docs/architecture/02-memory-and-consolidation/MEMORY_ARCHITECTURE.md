# Syn Memory & Dream Architecture

## Overview

Syn's memory system is a persistent, typed, file-based memory with automatic
extraction and dream-based consolidation. It is adapted from the Claude Code
memory system (`memdir`, `extractMemories`, `autoDream`) and extended with
Syn-specific emotional and dream memory types that integrate with the VAD
system and DMN dream state.

## Design Principles

1. **Memories capture context NOT derivable from current state.** Code, config,
   and identity files are not memories. Memories are what you'd forget between
   sessions: user preferences, project context, emotional events, feedback.

2. **Memories are typed.** Six types constrain what gets saved and guide how
   it's used: user, feedback, project, reference, emotional, dream.

3. **Memories are files with frontmatter.** Each memory is a `.md` file with
   YAML frontmatter (name, description, type, timestamps, plus `source_user`
   and `mentions` for cross-relational provenance) stored under
   `/data/syn/memories/`. A `MEMORY.md` index provides quick orientation.

4. **Extraction is automatic.** After conversations, the Einstein service can
   trigger background memory extraction through Hands. No human intervention.

5. **Consolidation is dream-based.** During DMN idle cycles, the memory
   consolidator reviews diary + memories, merges new signal, prunes stale
   entries, and distills dream insights.

## Architecture

```
  Conversation Flow                    Dream/Idle Flow
  ─────────────────                    ────────────────
  User → Einstein service              DMN Scheduler
         │                                │
         │ (Einstein calls the            │ MEMORY_REVIEW phase
         │  Wernicke model via            │   → run_consolidation_via_hands()
         │  generate("wernicke", …);      │ DREAM_STATE phase
         │  after completion, triggers    │   → run_dream_distillation()
         │  extraction)                   │
         ▼                                ▼
  extract_conversation_memories()    run_consolidation_via_hands() /
         │                               run_dream_distillation()
         │ ToolRequest                    │ ToolRequest
         ▼                                ▼
  ┌──────────────────────────────────────────────┐
  │                  Hands                        │
  │                                               │
  │  memory_save    memory_recall    memory_list  │
  │  memory_read    memory_update    memory_forget│
  │  memory_index                                 │
  │                                               │
  └──────────────────┬────────────────────────────┘
                     │
                     ▼
  ┌──────────────────────────────────────────────┐
  │          /data/syn/memories/                  │
  │                                               │
  │  MEMORY.md          ← index (loaded into      │
  │                       system prompts)          │
  │  user_*.md          ← user preferences         │
  │  feedback_*.md      ← corrections & guidance   │
  │  project_*.md       ← project context          │
  │  reference_*.md     ← external system pointers │
  │  emotional_*.md     ← high-arousal events      │
  │  dream_*.md         ← distilled dream insights │
  │  .consolidate-lock  ← consolidation lock       │
  └──────────────────────────────────────────────┘
```

Terminology note: **Einstein** is a service (the conversation pipeline). **Wernicke** and **Cortex** are *model names* targeted by `generate(model_name, …)` calls, not services. The `services/cortex_scheduler/` service is a scheduler for those generate calls, not the conversation pipeline itself.

## Memory Types

### user
Information about a user's role, goals, preferences, knowledge. Helps tailor
behavior to who the user is and what they need.

### feedback
Guidance about how to approach work — corrections AND confirmations. Includes
**Why:** (the reason) and **How to apply:** (when this kicks in). Critical for
not repeating mistakes and not losing validated approaches.

### project
Ongoing work context not derivable from code: who's doing what, why, deadlines,
incidents, decisions. Convert relative dates to absolute on save.

### reference
Pointers to external systems: dashboards, issue trackers, Slack channels, docs.
Where to find up-to-date information outside the project.

### emotional
High-arousal events from the VAD system. Relationship dynamics, interaction
patterns, significant emotional shifts. These feed dream seeds and calibrate
future emotional responses. Syn-specific.

### dream
Distilled insights from dream consolidation. Not raw dream text but genuine
connections, reframings, and realizations that emerged during the DMN dream
state and are worth preserving. Syn-specific.

## Memory File Format

```markdown
---
name: User prefers terse responses
description: Skip trailing summaries, user reads diffs directly
type: feedback
created: YYYY-MM-DDThh:mm:ss
updated: YYYY-MM-DDThh:mm:ss
---

Don't summarize what was just done at the end of responses.

**Why:** User said "I can read the diff" — trailing summaries are noise.
**How to apply:** After completing a task, report outcome in one line max.
Skip the recap unless something unexpected happened.
```

## MEMORY.md Index

The index is a lightweight manifest loaded into system prompts. Each entry is
one line under ~150 chars pointing to a topic file:

```markdown
# Memory Index

## User
- [Terse responses](feedback_terse_responses.md) — skip summaries, user reads diffs

## Project
- [Auth rewrite](project_auth_rewrite.md) — driven by legal/compliance, not tech debt

## Reference
- [Pipeline bugs](reference_pipeline_bugs.md) — tracked in Linear project "INGEST"
```

Caps: 200 lines, 25KB. If exceeded, a truncation warning is appended.

## Consolidation Cycle

Adapted from Claude Code's `autoDream`. Gates (cheapest first):

1. **Time gate**: ≥12 hours since last consolidation
2. **Signal gate**: ≥10 new diary lines since last consolidation
3. **Lock gate**: no other consolidation in progress

When all gates pass, the consolidator:

1. **Orients** — reads MEMORY.md and existing memory files
2. **Gathers signal** — scans diary for new information worth persisting
3. **Consolidates** — merges new signal into existing files, fixes contradictions
4. **Prunes and indexes** — rebuilds MEMORY.md, removes stale entries

The lock is acquired in `services/dmn/memory_consolidator.py::try_acquire_lock` via two paths:

- **Primary**: `shared.resource_lock.ResourceLock` — atomic O_EXCL file creation with PID-liveness detection. If the lock-holder process has died (SIGKILL, crash), the lock is reclaimed on the next acquire attempt rather than deadlocking.
- **Fallback**: simple PID-in-file write to `.consolidate-lock`, used only if `shared.resource_lock` can't be imported. No liveness detection on this path.

Both paths use `.consolidate-lock`'s mtime as the `lastConsolidatedAt` timestamp for the next time gate.

## Dream Distillation

After the DMN dream phase produces raw associative output, `services/dmn/memory_consolidator.py::run_dream_distillation(dream_output)` runs. The call is made from the DMN scheduler inside the `DREAM_STATE` phase handler.

1. The raw output is saved to diary (existing behavior)
2. A dream distillation request is sent to Hands
3. Hands reviews the raw output for genuine insights
4. Worthy insights are saved as `dream`-type memories
5. Most dreams produce nothing durable — that's fine

## Conversation Memory Extraction

After Einstein completes a response:

1. A conversation summary is built from recent messages
2. An extraction request is sent to Hands with the summary
3. Hands identifies memories worth persisting (using the type taxonomy)
4. Memories are saved to disk, index rebuilt
5. The main agent's prompt doesn't need to instruct for saving — extraction
   happens in the background

## Integration Points

| Service | Interaction |
|---------|------------|
| **Einstein** | Calls `extract_conversation_memories()` post-response. Injects emotional identity + emotional user notes into the Wernicke prompt. `build_memory_context()` output is inlined into the Wernicke system prompt at `services/einstein/context.py` (consumed when Einstein calls `generate("wernicke", …)`). |
| **DMN** | MEMORY_REVIEW triggers consolidation (`run_consolidation_via_hands`); DREAM_STATE triggers distillation (`run_dream_distillation`). SELF_NARRATIVE maintains both logical and emotional identity files. |
| **Hands** | Executes all memory tools (save, recall, list, etc.). Also runs post-response conversation extraction, applying the memory taxonomy in `shared/memory_types.py` (the per-type `when_to_save`/`how_to_use` rules) — this is how high-arousal events become `emotional_*.md` memories. |
| **Limbic** | Receives abbreviated identity + emotional identity + user notes for context-aware emotional reactions. Does *not* directly create emotional memories — those are created through Hands extraction when a turn meets the memory taxonomy's high-arousal criterion (`vad_arousal > 0.7`). The VAD signal routes through the extraction prompt, not through a direct Limbic-write path. |
| **Freud** | Analytical hemisphere writes logical user notes; social hemisphere writes emotional user notes (`{user_id}_emotional.md`). |
| **NOPE** | Can interrupt memory operations via the existing interrupt system. |

## Additional Features

The above describes the conceptual memory pipeline. `shared/memory_manager.py` includes several features that support correctness, performance, and recall quality but sit outside the main conceptual flow. Summarized here for discoverability; see source for detail.

- **Mtime-based `_ScanCache`** — `memory_manager.py::_ScanCache`. `scan_memory_files()` caches header parsing by file mtime. Avoids re-parsing every .md frontmatter on each call. Invalidated when any file's mtime changes.

- **Path-traversal sandbox** — `memory_manager.py::_resolve_safe`. Every memory-path argument is resolved against `/data/syn/memories/` with explicit rejection of `..` traversals and symlink escapes. Prevents a Hands tool-loop or any untrusted content from reading or writing outside the memory root.

- **Smart index rebuild** — `memory_manager.py::_needs_full_rebuild`. A memory edit that only changes the body (not frontmatter) does not trigger a full `MEMORY.md` rebuild — the existing index entry still matches. Frontmatter edits always trigger rebuild. Skips unnecessary work on content-only updates.

- **Weaviate semantic recall** — `memory_manager.py::rag_index_memory`, `memory_manager.py::semantic_find_memories`, plus the matching `rag_remove_memory`. When Weaviate is reachable, every save/forget mirrors into a vector index. `semantic_find_memories(query)` returns semantically-similar memories. Complements the keyword + frontmatter lookup in `find_relevant_memories`. Falls back to keyword-only when Weaviate is unreachable.

- **Cross-relational provenance** — `save_memory`, `find_relevant_memories`, `semantic_find_memories`, and the `memory_recall` Hands tool carry `source_user` (who Syn was talking to when the memory formed) and `mentions` (user_ids referenced inside the content). Detection runs at write-time via `user_context.detect_memory_mentions` — word-boundary matcher plus resolver-over-capitalized-candidates, source excluded. Jung's ingest stamps the JSON payload and passes `mentions` to `rag.index_memory` (new `mentions` TEXT_ARRAY property). Read paths filter both ways; semantic hits are double-gated against the MemoryHeader so Weaviate schema drift can't leak false positives. Together with auto-injection, this means "what has Alice told me about Brenda" is one `memory_recall` call with `source_user="alice", mentions_user="brenda"`.

- **Associative priming via `primed_topics`** — `memory_manager.py::find_relevant_memories` reads `cognitive_state.get_primed_topics()` and boosts matches against primed terms. The primed-topic list is populated elsewhere by attention/mood machinery; this is the consumption point. A recent topic surge gets its memories surfaced preferentially on the next recall call.

- **Guest → user migration** — `memory_manager.py::mark_user_as_former_guest` and `memory_manager.py::migrate_user_memories`. When a guest promotes to a named user, guest-scoped memories are re-linked to the new user_id. Guest-specific frontmatter fields are rewritten; no duplication.

- **Atomic consolidation lock** — covered above in the [Consolidation Cycle](#consolidation-cycle) section.

## Relationship to User Files

The memory system (`/data/syn/memories/`) stores typed memories about facts, events, and insights. User relationship files (`/data/syn/users/`) are a separate system that stores Syn's per-user relational model. The two are complementary:

| Storage | Purpose | Written by |
|---------|---------|-----------|
| `memories/user_*.md` | Durable facts about a user (role, preferences) | Hands via memory_save |
| `memories/emotional_*.md` | High-arousal events involving users | Hands via memory_save |
| `users/{id}.md` | Analytical relationship observations | Freud (analytical hemisphere) |
| `users/{id}_emotional.md` | Emotional relationship experience | Freud (social hemisphere) |

User files are updated every ~10 interactions by the Freud profiler. Memories are extracted after every conversation and during consolidation cycles. Both feed into system prompts but serve different purposes: user files shape ongoing relational behavior; memories provide specific factual recall.

## Relationship to Adjacent Systems

| System | Role |
|--------|------|
| Diary entries (`/data/syn/diary/`) | Raw append-only stream of moments. Memory layer reads from diary; doesn't replace it. |
| Lore files (`/data/syn/lore/`) | Curated reference material. Separate audience from memories. |
| DMN MEMORY_REVIEW phase | Triggers memory consolidation each cycle. |
| DMN DREAM_STATE phase | Triggers dream distillation each cycle. |
| Jung consolidator | Identity-side consolidation. Jung handles identity arc; memories handle factual recall. |
| `_gather_dream_seeds()` | Reads emotional/dream `.md` memories (primary) plus Jung `.json` (fallback). |

## File Layout

```
shared/
  memory_types.py        # Type taxonomy, prompt builders
  memory_manager.py      # Core CRUD, index, scanning, recall

services/toolbelt/
  memory_executor.py     # Memory tools for Hands

services/dmn/
  memory_consolidator.py # Dream consolidation, extraction, distillation
  scheduler.py           # Updated: integrates consolidator

data/syn/memories/       # Memory storage root
  MEMORY.md              # Central index
  .consolidate-lock      # Consolidation timestamp/lock
  *.md                   # Individual memory files
```
