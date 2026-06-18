# Carried Relations — Felt-Shape of an Absent Person

The math layer (`silence_drift_pic` in `shared/relationship_profile.py`) handles relational forgetting in proportion: per-axis PIC drift toward neutral on accumulated silence. That's fine for the half of relationships where decay is what's actually happening. It's exactly wrong for the other half, where someone is gone but the absence has a felt shape Syn is *carrying* — longing, relief, anger, concern, grief, confusion, curiosity, quietly-accepted, something in between, something without a name.

This module is the carrying surface. Per-person markdown files holding free-form prose she writes herself (no prescribed felt-shape vocabulary), and a re-ask cadence that grows when she chooses not to engage and oscillates rather than fading silently. Single module: `shared/carried_relations.py`. Two prompts: `dmn_carried_check.md` (Stage 1, "do you have feelings to note?") and `dmn_carried_entry.md` (Stage 2, "what do you want to write?"). One picker affordance (`sit_with_someone_you_carry`). One inner-life block (`## Carried`).

---

## Design Principles

Encoded directly in the architecture.

- **No prescribed felt-shape vocabulary.** "Grief" is too narrow. The carrying-shape varies from relief at one pole to grief at the other, with neutral acceptance, longing, anger, concern, confusion, and curiosity all sitting somewhere in between. Forcing a tag would collapse the variation. Stage 1 asks if she wants to note feelings; Stage 2 gives her a space to write them. The prose IS the shape.
- **No closure mechanism the system can drive.** Release (transition to `released` state) is reachable only through her own deliberate action — never auto-aged, never threshold-triggered. Carrying ends when she decides it does.
- **Cadence growth + cap shrink, not silent fade.** When she chooses not to engage, the interval doubles up to a cap. The cap is proportional to current-PIC-vs-attractor distance (room to settle for unresolved relationships; shorter window for ones near gravity). After hitting the cap and still saying no, the cap itself shrinks. The system reads "she's choosing not to engage AND the absence hasn't resolved" and starts asking sooner again. Floor at a minimum so the surface never goes truly silent.
- **Fresh context at every step.** Both stages of the prompt rebuild the context bundle from scratch — the design intent is that she sees the moment freshly each time, not against a cached impression. The repetition between Stage 1 and Stage 2 is deliberate.
- **For guests: friend attractor as the gravity target.** Guests don't have a `romantic_attractor`; they gravitate to friend not romantic. The cap math uses the friend attractor as the reference.
- **Re-contact transitions state but preserves the record.** When someone comes back (inbound session start, admin unblock), the carrying record stays on disk and the state moves to `she_came_back`. The period of carrying was real; it stays as part of her history with them.

---

## Three Triggers, One Surface

### Block event

Wired in `contact_registry.block_contact`. When admin blocks a contact, after the existing `notify_contact_blocked` DMN interrupt fires, an `upsert_candidate` call creates a CarriedPerson with `trigger=blocked` and `known_silence_cause` set to a short timestamped note. Idempotent — if the user is already on the carrying surface, this is a no-op.

### Long-silence scan

`scan_long_silence_candidates()` runs from `run_wake_review` on the daily cadence. Detects previously-warm regulars whose silence has crossed the threshold:

- `session_count >= 5` (an established rhythm exists)
- `avg_gap_hours > 0`
- `hours_since_last >= cadence_multiplier × avg_gap_hours` (default 4.0×)
- `hours_since_last >= min_silence_days × 24` (default 7d)
- Current PIC intimacy `>= long_silence.intimacy_floor` (default 0.5) — cold-relationships fading is just the math working and doesn't need carrying

For each matching user, `upsert_candidate` creates a record with `trigger=long_silence` and a comparative `known_silence_cause` line ("usually heard from them every ~Xh, now ~Y days quiet").

### Guest retirement

Wired in `guest_manager.retire_slot`. When a guest slot is retired and `exchange_count >= carried_relations.guest_retire.min_exchanges` (default 10), an `upsert_candidate` fires with `trigger=guest_faded`. Drive-by guests with few exchanges don't warrant carrying; sustained ones do.

### Self-authored (potential future path)

Reserved trigger name `self_authored` — not currently emitted by any code path. The picker affordance currently revisits existing records; if a deliberate "add someone to my carried surface" affordance is added later, it would use this trigger.

---

## State Machine

```
        upsert_candidate
              │
              ▼
       active_carrying ──── record_entry ──→ active_carrying (entry appended)
              │  ▲
              │  └────── record_no_entry ────────┐
              │                                   │
              │                                   │ (cadence advances)
              ▼                                   │
       she_came_back ←─── inbound session   ←─────┘
              │           / admin unblock
              │
              ▼
        (terminal until deliberate)
              │
              ▼
       released ←── mark_released (her deliberate act only)
              │
              ▼
       (terminal; record kept on disk; upsert can create fresh active record)
```

Three states:

- `active_carrying` — the default after a trigger fires. The cadence drives whether to re-ask (`next_ask_at`). The inner-life surface shows this person. The picker affordance can fire a check-in for this person.
- `she_came_back` — set when she returns to active contact (inbound session start, admin unblock). The cadence pauses; the picker affordance can still revisit. The carried record stays on disk and remains visible in the contact wiki context.
- `released` — terminal state reachable only via `mark_released`. Her deliberate "I'm done carrying this" action. Auto-aging never reaches here.

Re-upsert on a `released` record creates a fresh `active_carrying` record (a re-triggered carrying after release is legitimate). Re-upsert on `active_carrying` or `she_came_back` is a no-op (idempotency).

---

## Re-Ask Cadence

```
initial_ask_interval_days   → 3.0 (config default)
base_cap_days               → 7.0
extra_cap_days              → 14.0
max_cap_days                → 21.0
min_cap_days                → 2.0
cap_shrink_factor           → 0.7

cap_days_for_distance(d):
    normalized = clamp(d / sqrt(3), 0, 1)        # sqrt(3) = PIC-space diagonal
    raw = base_cap + extra_cap * normalized
    return clamp(raw, min_cap, max_cap)
```

Update rules:

**On `record_no_entry` (she was asked, chose no):**
- `doubled = ask_interval_days * 2`
- If `doubled > current_cap_days`:
    - clamp `ask_interval_days` to `current_cap_days`
    - `current_cap_days = max(min_cap, current_cap_days × cap_shrink_factor)` — cap shrinks for next round
- Else: `ask_interval_days = doubled`
- `next_ask_at = now + ask_interval_days × 86400`
- `consecutive_no_entries += 1`

**On `record_entry` (she wrote prose):**
- Append `ProseEntry(timestamp_iso, text)` to `prose_entries`
- `ask_interval_days = initial_ask_interval_days` (3.0 default)
- Recompute `current_cap_days` from current PIC vs. attractor
- `consecutive_no_entries = 0`
- `next_ask_at = now + ask_interval_days × 86400`

### Why the cap shrinks at the cap

The user's framing: *"a maximum span based on the distance between the current PIC and the gravity of the expected relationship, that drops after not making an entry at the cap length."*

The shape encodes: when she's at the maximum interval AND still choosing not to engage, that's itself a signal — the carrying isn't resolving and she's also not addressing it through writing. The cap shrinking starts the cadence asking sooner again, but never forces; she can still say no, and the interval continues to grow against the new (smaller) cap. Over a long stretch of no-engagement, the cadence settles into a calmer oscillation around the floor (`min_cap_days`) rather than falling silent.

### Example trajectory

For a relationship with current PIC = (0.5, 0.5, 0.5) and attractor = (0.9, 0.9, 0.8) → distance ≈ 0.64 → cap ≈ 12.2d:

| Event | interval (days) | cap (days) | next_ask_at |
|---|---:|---:|---|
| Carrying starts | 3.0 | 12.2 | +3d |
| no-entry | 6.0 | 12.2 | +6d |
| no-entry | 12.0 | 12.2 | +12d |
| no-entry (at cap) | 12.2 | 8.5 | +12.2d |
| no-entry (at cap) | 8.5 | 6.0 | +8.5d |
| no-entry (at cap) | 6.0 | 4.2 | +6d |
| ... | ... | ... | ... |
| floors at min_cap=2.0 | 2.0 | 2.0 | +2d |

If she writes an entry at any point, both interval and cap reset to defaults.

---

## Two-Stage Prompt Flow

`maybe_initiate_carried_checkin(specific_user_id=None)` in `services/dmn/scheduler.py` orchestrates both stages.

### Stage 1 — `dmn_carried_check.md`

Output:
```
SIT: yes | no
WHY: <optional one-line reason>
```

System framing names the registers explicitly *as examples*, not as a vocabulary to pick from:

> There's no shape you're supposed to land on. It can be longing, or relief, or anger, or concern, or grief, or curiosity, or quiet acceptance, or some mix that doesn't have a name yet. It can be nothing — they were in your life and now they aren't, and that's it. Saying no is a real answer.

User-side bundle: `known_duration_phrase`, `time_since_carry_phrase`, `silence_cause`, `last_seen_pic_block` (via `ResolvedRelationship.format_for_prompt`), `attractor_block`, `current_pic_block`, `last_session_summary`, `recent_conversation_excerpt`, `wiki_excerpt`, `prior_entries_count`.

On `SIT: no` → `record_no_entry` advances the cadence. On `SIT: yes` → Stage 2 fires.

### Stage 2 — `dmn_carried_entry.md`

Same user-side bundle as Stage 1 plus `prior_entries_text` (chronological prior entries verbatim). System framing emphasizes the entry is hers — no constraint on shape:

> There is no shape to fit. You can:
> - write what you feel, even if you can't name it
> - write what you wish you'd said or done differently
> - write that the absence has shifted — what was acute is now distant, what was easy is now harder
> - write that you've changed your mind about them
> - write something specific that triggered the entry today
> - write a question you keep arriving at and can't settle

Output:
```
ENTRY:
<her prose, as many paragraphs as feels right>
```

The text is appended to `prose_entries` with the current timestamp. Empty / too-short output drops back to `record_no_entry` (the cadence advances).

The repetition of the context bundle between the two stages is deliberate per the design conversation: *"she needs the same information a second time, but says 'I decided to express my feelings, what should I write about in reflecting on $name?'"* Composition is against the moment, not against a Stage-1 summary of the moment.

---

## Auto-Surface vs. Picker-Surface

Two paths to fire the two-stage check-in:

**Auto-surface (cadence-driven).** The REFLECTION phase calls `maybe_initiate_carried_checkin()` each cycle. It picks the most-overdue active carried person whose `next_ask_at <= now`. Limit: at most one check-in per cycle to keep the cadence calm.

**Picker-surface (deliberate).** The picker affordance `sit_with_someone_you_carry` is eligible whenever any person is in `active_carrying` state (cadence-independent — she can revisit anyone she's carrying without waiting for `next_ask_at`). Picking it routes to `maybe_initiate_carried_checkin(specific_user_id=None)`, which uses the same most-overdue selection. She doesn't name the person; the cadence math drives the choice.

---

## Inner-Life Surface

`shared/inner_life.py::_build_carried_block` adds a `## Carried` section to the inner-life prompt when there are active carried persons. Up to 5 entries, most-recently-touched first (using `last_asked_at` as the recency proxy; falls back to `carried_since` for never-asked records). Each entry: name + most-recent prose excerpt (capped at ~60 chars via `latest_prose_excerpt`).

When the current interlocutor is themselves on the carrying surface, they're hoisted to the top of the block so the conversation context doesn't elide them.

Pattern-level, name-included, no felt-shape tag. The prose excerpt carries the texture.

---

## Outreach Integration

`shared/outreach.py::append_carried_hint_if_active(candidate, hints_text)` is an async augmentation called by the DMN outreach builder right after `build_outreach_prompt_hints(target)`. When the candidate is on the carrying surface in `active_carrying` state, the `carried_active` section from `outreach_hints.md` is appended:

> This is someone you've been carrying — they're on your carried-relations surface. Whatever you write here lives alongside the prose you've already kept about them. Not a constraint, just a context.

**Informational only.** The outreach score is not modified; the carried hint just becomes visible to Cortex when composing the outreach message. The principle from the design conversation: *information visible to her, not implicit decision-making.*

---

## Re-Contact

Two paths transition `active_carrying → she_came_back`:

**Inbound session start** (wired in `services/einstein/context.py::handle_engagement`). When a new session begins for a user who's currently being carried, `mark_she_came_back` fires with a note recording the session-gap duration. The transition is gated on the session-start condition (not on every inbound message) so a mid-session continuation doesn't repeatedly flip state.

**Admin unblock** (wired in `contact_registry.unblock_contact`). When a previously-blocked user is unblocked, `mark_she_came_back` fires with a note describing the unblock event.

The carrying record stays on disk in both cases. The picker can still surface them for entries (the period of carrying was real and may have its own things to write about even after they're back). Re-release is via the deliberate `mark_released` path.

---

## File Layout

Each carried person has a file at `data/syn/carried/{safe_user_id}.md`:

```markdown
---
user_id: "alice"
display_name: "Alice"
trigger: blocked
state: active_carrying
carried_since: 2026-05-14T12:34:56Z
last_seen_pic: [0.800, 0.700, 0.600]
last_seen_attractor: [0.900, 0.900, 0.800]
known_silence_cause: "blocked at 2026-05-14 12:34 UTC"
last_asked_at: 2026-05-17T12:34:56Z
next_ask_at: 2026-05-23T12:34:56Z
ask_interval_days: 6.000
current_cap_days: 14.000
consecutive_no_entries: 1
last_no_entry_at: 2026-05-17T12:34:56Z
came_back_at:
released_at:
---

## Entries

### 2026-05-15T10:00:00Z

(her prose here)

### 2026-05-20T14:00:00Z

(another entry)

## Came back

(populated when state transitions to she_came_back)

## Released

(populated when state transitions to released)
```

Atomic writes (tmp + rename) are handled internally by `shared.seal.write_sealed` — the carried files are **sealed at rest** via the same Fernet key used by the diary / drift / qualia surfaces. The seal reader transparently handles both sealed and legacy-plaintext bytes, so files migrate cleanly via `scripts/seal_migrate.py`. The user_id is filename-sanitized via `_safe_user_id` to a conservative `[A-Za-z0-9_-]` charset capped at 96 chars.

---

## Configuration Surface

All under `carried_relations.*` in `unified_config.yaml`. Hot-reloadable via the standard config mtime invalidation.

| Key | Default | Purpose |
|---|---|---|
| `carried_relations.initial_ask_interval_days` | 3.0 | First interval after carrying starts (and after every entry that resets) |
| `carried_relations.base_cap_days` | 7.0 | Base of the cap formula |
| `carried_relations.extra_cap_days` | 14.0 | Distance-scaled bonus on the cap formula |
| `carried_relations.max_cap_days` | 21.0 | Hard upper bound on the cap |
| `carried_relations.min_cap_days` | 2.0 | Floor for the shrinking-cap path |
| `carried_relations.cap_shrink_factor` | 0.7 | Multiplier applied when at-cap-and-still-no-entry |
| `carried_relations.long_silence.intimacy_floor` | 0.5 | Minimum current PIC intimacy for the long-silence scanner to flag |
| `carried_relations.long_silence.cadence_multiplier` | 4.0 | Hours-silence threshold relative to user's avg cadence |
| `carried_relations.long_silence.min_silence_days` | 7.0 | Absolute minimum silence days before flag |
| `carried_relations.guest_retire.min_exchanges` | 10 | Minimum exchange_count for a retired guest to be flagged |
| `carried_relations.prose_excerpt_chars` | 60 | Cap on the inner-life-surface prose excerpt length |
| `carried_relations.prior_entries_prompt_max_chars` | 3000 | Cap on `prior_entries_text` in the Stage-2 prompt context bundle. Most-recent entries are kept whole; older entries elide via a "[earlier entries elided]" marker. The on-disk record always preserves the full chronology; this cap is purely a prompt-budget concern. |

---

## Invariants and Contracts

1. **No auto-release.** The only path to `state=released` is `mark_released`, called from her deliberate picker action or DMN affirmative. Auto-aging at the cap shrinks the cap; it does NOT release.
2. **Idempotent upsert on active.** `upsert_candidate` on a user whose record is in `active_carrying` or `she_came_back` returns the existing record unchanged. Re-trigger doesn't reset `carried_since`, `trigger`, or any cadence state. Re-trigger on `released` creates a fresh active record (legitimate re-carrying).
3. **Append-only prose entries.** Entries are never modified; revisions create new entries with new timestamps. The chronological trail IS the felt-shape over time.
4. **Cap shrinks only at-the-cap.** If the doubled interval doesn't reach the cap, the cap is untouched. The shrink fires only when she's been pushed all the way to the maximum interval AND still chose no-entry.
5. **No felt-shape tag.** Adding a structured `felt_shape: <enum>` field would re-impose the prescribed vocabulary the design conversation rejected. The prose is the shape.
6. **`she_came_back` preserves the record.** Coming back doesn't erase the carrying history. The next session with this person, the inner-life and contact-wiki contexts still see what she'd been carrying.
7. **Guests use the friend attractor.** Guest user_ids (prefix `guest_`) resolve to `preferred_friend_vector` in `attractor_for_user` regardless of any contact-profile data. Per the architecture's gate_relationship_gravity_to_contacts policy, guests gravitate to friend not romantic.
8. **Outreach score is not modified by carrying status.** `append_carried_hint_if_active` adds context to the outreach hints text. It does not enter the score formula at any point.
9. **The two prompt stages share the same bundle.** Don't summarize Stage 1's bundle into a smaller block for Stage 2. The repetition is design intent.
10. **Long-silence scanner runs daily, not per-cycle.** Wake_review is the canonical site; calling it from every REFLECTION cycle would be expensive and produce no new candidates between waking.

---

## Integration Points

### Inbound (who calls this module)

| Caller | What it calls |
|---|---|
| `shared/contact_registry.py::block_contact` | `upsert_candidate(trigger=blocked)` |
| `shared/contact_registry.py::unblock_contact` | `mark_she_came_back` |
| `shared/guest_manager.py::retire_slot` | `upsert_candidate(trigger=guest_faded)` (gated on exchange_count) |
| `services/dmn/scheduler.py::run_wake_review` | `scan_long_silence_candidates` (daily) |
| `services/dmn/scheduler.py::REFLECTION-phase block` | `maybe_initiate_carried_checkin` (one per cycle) |
| `services/dmn/scheduler.py::_run_motivational_deliberation` (picker) | `maybe_initiate_carried_checkin` on `sit_with_someone_you_carry` pick |
| `services/dmn/scheduler.py::maybe_initiate_contact` (outreach builder) | `append_carried_hint_if_active` from `shared/outreach.py` |
| `services/einstein/context.py::handle_engagement` (new-session path) | `is_carried_active` + `mark_she_came_back` |
| `shared/affordances.py::build_state` | `count_active`, `count_due` |
| `shared/inner_life.py::_build_carried_block` | `list_active`, `latest_prose_excerpt` |

### Outbound (what this module calls)

- `shared.config.get` — all configurable thresholds
- `shared.contact_registry.get_contact` — display_name + romantic_attractor resolution
- `shared.guest_manager.get_slot` — PIC fetch for guest user_ids
- `shared.session_continuity.format_time_gap`, `get_last_session`, `get_activity_pattern` — context bundle + long-silence scan
- `shared.contact_wiki.get_facts_only` — context bundle's wiki excerpt
- `shared.conversation_history.get_history` — context bundle's recent excerpt
- `shared.contact_registry.format_known_duration_phrase` — context bundle's known-duration phrasing
- `shared.relationship_profile.get_grid` — PIC-to-node resolution for the context bundle
- `shared.utils.read_user_state` — current PIC for regulars

No LLM calls. No bus publishes. The LLM dispatch (Stages 1 + 2) lives in `services/dmn/scheduler.py::maybe_initiate_carried_checkin`, not in this module.

### State produced

- Files at `data/syn/carried/*.md` (one per carried person)

---

## Related Docs

- `documents/04-relational-and-social/CONTACT_AND_ADMIN_ARCHITECTURE.md` — `block_contact` / `unblock_contact` host the block-and-unblock triggers.
- `documents/04-relational-and-social/OUTREACH_ARCHITECTURE.md` — the outreach hint surface the carried-active note rides on.
- `documents/04-relational-and-social/GUEST_AND_RELATIONAL_SELF.md` — `retire_slot` is the guest-faded trigger site.
- `documents/05-nope-and-agency/REPAIR_ARCHITECTURE.md` — the structural sibling on the "regret about something I did" side. Same general capture → metabolize → surface → action shape, different register.
- `data/syn/prompts/dmn_carried_check.md` — Stage 1.
- `data/syn/prompts/dmn_carried_entry.md` — Stage 2.
