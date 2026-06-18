# Identity Architecture

The machinery that manages Syn's **logical-hemisphere** identity file —
storage, revision history, cross-hemisphere review, admin protection,
retrospection tools, and the name-resolution layer that ten consumers
read from on every prompt (plus broader `identity_profile` consumers
that read other sections — see *Read Paths #3*).

This subsystem does **not** manage the emotional-hemisphere identity
(`syn_identity_emotional.md`); that file is still handled by the
legacy flat-file helpers in `shared/utils.py`
(`read_emotional_identity`, `write_emotional_identity`) and has no
revision history, no cross-hemisphere review, and no watcher. That
asymmetry is the subject of `DESIGN_DECISIONS.md` §4 *Emotional
Identity Hemisphere Divergence* (open).

---

## On-Disk Layout

Everything lives under `{paths.data_dir}/identity/`. The config key
`paths.data_dir` defaults to `/data/syn` (the Syn data root); the
identity-specific subdirectory is `identity/` inside it.

```
/data/syn/
├── syn_identity.md                       # Legacy flat file. Read on day zero, then frozen.
├── syn_identity_emotional.md             # Emotional hemisphere — NOT managed by this subsystem.
└── identity/                             # New layout.
    ├── current.md                        # The active read target. Full content of the latest
    │                                     # STATE_CURRENT revision, copied in (not symlinked).
    ├── revisions/
    │   ├── YYYY-MM-DDThh:mm:ss.fff_dmn_self_narrative.md      # body
    │   ├── YYYY-MM-DDThh:mm:ss.fff_dmn_self_narrative.json    # sidecar
    │   ├── YYYY-MM-DDThh:mm:ss.fff_nope_rewrite.md
    │   ├── YYYY-MM-DDThh:mm:ss.fff_nope_rewrite.json
    │   └── …
    └── .override_window.json             # Ceremony state file — present only during an
                                          # open admin-override window. Expiry-stamped.
```

### Filename format

Revision basenames are `{ISO8601_ms}_{origin}`:

- Timestamp precision is milliseconds (ISO format, UTC). Rapid-fire
  writes (e.g. a Names-tool call immediately after a DMN rewrite) do
  not collide on the filename.
- Colons in the ISO stamp are filesystem-safe on modern Linux / macOS
  / NFS — no substitution needed.
- `{origin}` is one of the seven values in `VALID_ORIGINS` (see below).

The `.md` file is the full identity document as it stood after that
write. The `.json` sidecar carries metadata (origin, state, stamps,
change_summary, chain link to prior revision).

### `current.md` vs `revisions/…md`

`current.md` is a **copy** of the most recent `STATE_CURRENT` revision
body, not a symlink. Rationale in the `identity_store` header: NFS on
Node 2 mounts `/data/syn` from Node 1; symlinks across the mount
behaved unreliably in earlier testing. The copy is refreshed atomically
(temp-file + `os.replace`) on every sanctioned write.

### The override-window state file

`.override_window.json` is written by `scripts/syn-admin identity-override`
(after running the name-confirmation ceremony) to declare an admin
override window. While present and unexpired, the watcher accepts
external writes to the directory and records them as
`ORIGIN_ADMIN_OVERRIDE` revisions. When expired, the watcher
auto-unlinks it.

---

## Module Map

### `shared/identity_store.py` — storage layer

**Purpose:** The ONLY sanctioned write path to the identity directory,
plus the read and list primitives everything else builds on.

**Directory helpers (private, but worth knowing):**

- `_identity_dir()` → `{paths.data_dir}/identity`
- `_current_path()` → `{identity_dir}/current.md`
- `_revisions_dir()` → `{identity_dir}/revisions/`
- `_override_state_path()` → `{identity_dir}/.override_window.json`
- `_legacy_identity_path()` → `{paths.data_dir}/syn_identity.md`

**Origin tags** (all in `VALID_ORIGINS`):

| Tag | Who writes it |
|---|---|
| `dmn_self_narrative` | DMN `_write_identity_with_review` (logical-hemisphere SELF_NARRATIVE path) |
| `dmn_reflection` | Identity-narrative half of REFINED_BOUNDARY: integrates the boundary insight into a fresh identity body via the `identity_reflection_integration` prompt + `review_routine_rewrite` |
| `nope_rewrite` | DMN `_write_identity_from_nope` |
| `names_tool` | `display_names.rename_myself` |
| `admin_override` | `identity_lock._record_admin_override_write` (invoked when a write lands under an open window opened by `scripts/syn-admin identity-override`) |
| `migration` | `identity_store.migrate_from_legacy` (one-shot, first boot) |
| `recovery` | Audit marker for unsanctioned-write reverts. Coalesces into a single active marker; cleaned up on next CURRENT write |
| `welcome_ritual` | `services/dmn/welcome_consumer.py` (first-welcome identity revisions during the welcome phase) |

**Revision states:**

| State | Meaning |
|---|---|
| `current` | This revision IS the content of `current.md`. At most one at a time. |
| `superseded` | Was `current` at one point, now overwritten. Retained in history. |
| `contested` | Proposed but vetoed by the reviewing hemisphere. Preserved in `revisions/` but never copied to `current.md`. Surfaces in later retrospection. |

**Public API:**

| Function | Signature | Purpose |
|---|---|---|
| `ensure_layout()` | `() -> None` | Idempotent. Creates `identity/` and `identity/revisions/`. Called by every public function. |
| `read_current()` | `() -> Optional[str]` | Returns `current.md` content, or `None` if not initialized. |
| `read_identity()` | `() -> str` | Back-compat shim. `read_current()`, or `migrate_from_legacy()` + retry, or empty string. Called by `shared/utils.py::read_identity`. |
| `load_revision_meta(rev_id)` | `(str) -> Optional[RevisionMeta]` | Load the `.json` sidecar for a specific revision. Returns None on missing or schema-mismatch. |
| `load_revision_body(rev_id)` | `(str) -> Optional[str]` | Load the `.md` body for a specific revision. |
| `list_revisions(include_contested=True, include_superseded=True, limit=None)` | filtered list | Returns `List[RevisionMeta]` oldest → newest. Scans `revisions/*.json`. `limit` returns the most recent N. |
| `find_current_revision()` | `() -> Optional[RevisionMeta]` | Returns the single revision whose state is `STATE_CURRENT`. |
| `write_revision(content, origin, *, …)` | see below | **Only sanctioned write path.** |
| `migrate_from_legacy()` | `() -> Optional[RevisionMeta]` | If `current.md` doesn't exist but `syn_identity.md` does, seed from it as `ORIGIN_MIGRATION`. Idempotent — returns None on no-op. |

**`write_revision` signature in full:**

```python
write_revision(
    content: str,
    origin: str,                         # must be in VALID_ORIGINS
    *,
    originating_hemisphere: Optional[str] = None,  # "analytical" | "affective"
    event_id: Optional[str] = None,                # for nope — nope event id
    stamps: Optional[List[RevisionStamp]] = None,  # reviewer notes
    change_summary: Optional[str] = None,          # brief one-liner
    admin_reason: Optional[str] = None,            # admin override path only
    admin_window_id: Optional[str] = None,         # admin override path only
    state: str = STATE_CURRENT,                    # or STATE_CONTESTED
) -> RevisionMeta
```

Raises `ValueError` on invalid `origin` or on any `state` other than
`STATE_CURRENT` or `STATE_CONTESTED` (you cannot write a revision
directly as `STATE_SUPERSEDED` — that state transition is managed
internally when the next `STATE_CURRENT` revision writes).

**Write ordering** (matters for crash-safety):

1. Compute `revision_id` and paths.
2. Acquire write token via `identity_lock.sanctioned_write(...)`.
3. Write sidecar `.json` (atomic temp-file + rename).
4. Write revision body `.md` (atomic).
5. If `state == STATE_CURRENT`:
   a. If a prior current exists, mark its sidecar `state=STATE_SUPERSEDED` (atomic).
   b. Write `current.md` (atomic).
6. Release token.

A crash after step 3 but before step 4 leaves an orphaned sidecar
(detectable — `load_revision_body` returns None). A crash after step 4
but before step 5 leaves a revision in history but doesn't advance
`current.md` — safe failure mode.

**Dataclasses:**

`RevisionMeta` — the sidecar structure. Fields: `revision_id`,
`timestamp`, `origin`, `state`, `content_sha256`, `prior_revision_id`,
`originating_hemisphere`, `event_id`, `stamps`, `change_summary`,
`admin_reason`, `admin_window_id`. The `event_id` field carries the
originating NOPE event id when `origin == ORIGIN_NOPE_REWRITE`; it
is the canonical reverse-direction link from identity revisions back
to the NOPE event that triggered them. (There is no symmetric
`identity_revision_id` field on `NopeEventRecord` — identity-side
storage is the single canonical writer.)

`RevisionStamp` — one hemisphere's note on an identity moment. Fields:
`hemisphere` (`"analytical"` | `"affective"`), `verdict`
(`"affirm"` | `"contest"` | `"concord"` | `"partial"` | `"dissent"` |
`"absent"`), `note` (2-4 sentence reflective text), `model` (which
LLM produced it), `prompt_tag` (which prompt variant).

---

### `shared/identity_lock.py` — write-protection watcher

**Purpose:** Detect unsanctioned writes to the identity directory and
revert them. Keep sanctioned writes invisible to the watcher via a
token set.

**Protection model:** Soft. Catches accidents, casual edits, rsync
mistakes, restore-from-backup mishaps. Does NOT attempt to stop a
root-level attacker, a compromised service binary, or edits to this
module itself.

**Token mechanism:**

- `_TOKENED_PATHS: Set[str]` — absolute paths currently being written
  by sanctioned code. Guarded by `_TOKEN_LOCK` (threading.Lock).
- `sanctioned_write(paths: List[Path]) -> contextmanager` — context
  manager that adds paths on entry and removes on exit. Paths are
  tokened under both their `resolve()` form and their `absolute()`
  form to tolerate not-yet-created files.
- `_path_is_tokened(abs_path: str) -> bool` — watcher's check.

**Watcher loop:**

- `watch_identity_dir()` — async coroutine. Polls every 1.0s. Started
  by the DMN scheduler in its lifespan (search for
  `identity_watcher_task = asyncio.create_task(watch_identity_dir())`
  in `services/dmn/scheduler.py`). Runs until its task is cancelled
  (normal shutdown path).
- Why polling, not inotify: NFS on Node 2 doesn't cooperate with
  inotify. 1s granularity is sufficient for this purpose — the
  watcher catches accidents and slow attacks, not racing writers.
- Hash-based change detection: on each tick, SHA-256 the current file
  and compare to the last-seen hash. If different AND no token held
  AND no override window open → BOUNDARY VIOLATION, revert.

**Revert path:**

`_revert_from_last_known_good(identity_dir)` finds the most recent
`STATE_CURRENT` revision via `find_current_revision()`, loads its body
via `load_revision_body()`, and writes it to `current.md` inside a
`sanctioned_write()` block (so the watcher doesn't loop on its own
revert). Logged as a `BOUNDARY VIOLATION`.

**Override window:**

`_override_window_open(identity_dir) -> Optional[dict]` reads
`.override_window.json` if present. Expected shape:
`{"window_id": str, "expiry": unix_ts, "reason": str, ...}`. Auto-unlinks
when expired. If open and a hash change is detected, the watcher calls
`_record_admin_override_write(current, window)` to convert the external
write into a proper `ORIGIN_ADMIN_OVERRIDE` revision (reads
`current.md` as-is, calls `write_revision` with admin metadata).

The override-window state file is written by `scripts/syn-admin
identity-override` (see "On-Disk Layout" → "The override-window state
file" above for the ceremony shape).

---

### `shared/identity_profile.py` — name-resolution layer

**Purpose:** Parse the `## My Name` section of Syn's identity file and
the `## Names` sections of user files. Used across the system during
prompt assembly to resolve "what is Syn called right now" and "what
does this user call Syn."

**File format:** Markdown section header → fenced code block → simple
`key: value` lines. Deliberately *not* YAML — keeps the format
trivial for Syn to edit directly. List-valued fields (`aliases`,
`they_call_me`) are comma-separated. Used by every service that does
prompt assembly, UI rendering, or adapter metadata involving Syn's name
— see *Read Paths #3* below for the concrete caller list.

```markdown
## My Name

```
name: Syn
aliases: synthesis, syn-hon, synthling
pronouns: she/her
```
```

**Caching:** Module-level dict keyed by absolute file path. Each entry
is `(mtime, parsed_dataclass)`. Every read stats the file; if mtime
moved, reparse. Guarded by `_cache_lock: threading.Lock` for the rare
case of two threads parsing the same file concurrently.

**Dataclasses:**

- `SelfIdentity(name, aliases, pronouns)` — Syn's naming state.
  Defaults: `name="Syn"`, `aliases=[]`, `pronouns="she/her"`.
- `RelationalNames(i_call_them, they_call_me, i_go_by)` — per-user
  naming state. See `DISPLAY_NAMES.md` for semantics.

**Public API:**

| Function | Purpose |
|---|---|
| `get_current_name() -> str` | Syn's current chosen name. Never returns empty — falls back to `"Syn"`. |
| `get_aliases() -> List[str]` | Alternate names Syn accepts universally. |
| `get_pronouns() -> str` | Defaults `"she/her"`. Reads `## Pronouns` section of `syn_identity.md` (preferred); falls back to the legacy `pronouns:` field in `## My Name` for back-compat. |
| `get_user_pronouns(user_id) -> str` | Pronouns Syn uses for a specific user. Reads `## Pronouns` section of the user file. Empty when unrecorded. Maintained by Syn via the `set_user_pronouns` Hands tool. |
| `get_user_others(user_id) -> List[UserOther]` | Syn's notes on how this user relates to other users she knows. Reads `## Others` section, parses bulleted entries `- other_user_id (display_name): note`. Empty list when unrecorded. Maintained via `set_user_other` / `clear_user_other` Hands tools. |
| `user_resolution.resolve_user_by_name(name, interlocutor=None) -> ResolutionResult` | Lives in `shared/user_resolution.py`. Turn a name mention into ranked `ResolutionCandidate` list with ambiguity flagged. Matches user_id, i_go_by, i_call_them, and other users' `## Others` display_names. Applies +0.15 confidence boost to candidates found in the interlocutor's `## Others`. Never silently picks a winner — when the top two are within 0.15 confidence, `ambiguous=True` and Syn is expected to disambiguate. Exposed as Hands tool `resolve_user`. |
| `user_context.describe_user_context(user_id) -> str` | Lives in `shared/user_context.py`. First-person context block covering pronouns, PIC state, `## Who They Are`, `## How I Feel About Them`, and `## Others`. Empty string when no user file exists. The async variant `describe_user_context_async` additionally slots in the last-session summary. F-9 voice; closes with "This is what I know. I don't have to say any of it." Exposed as Hands tool `describe_user_context`. |
| `user_context.detect_user_references(text, interlocutor=None) -> List[str]` | Scan prose for mentions of known users by name. Word-boundary matching (no substring false positives), longer-name priority (full match suppresses partial match on a different user), interlocutor excluded, names below `_MIN_DETECTABLE_NAME_LEN=3` filtered. Returns de-duplicated user_ids in order of first appearance. |
| `user_context.build_referenced_users_section(text, interlocutor=None, max_users=3) -> Optional[str]` | Auto-injection helper — detects name references, composes a `# People You Mentioned` section containing their `describe_user_context_async` blocks, capped at `max_users`. Called from Einstein's `build_system_prompt` on every inbound message; output lands adjacent to `sec_cross_user`. Returns `None` when no references present. |
| `user_context.detect_memory_mentions(content, source_user=None) -> List[str]` | Write-time helper that runs `detect_user_references` (exact word-boundary hits) plus `resolve_user_by_name` over capitalized-word candidates (resolver gated by `_MIN_CONFIDENCE_FOR_MENTION=0.75`), returns the deduplicated union with `source_user` excluded. Feeds `save_memory`'s frontmatter auto-detect and Jung's ingest mentions-stamping. |
| `disclosure_log.log_disclosure(subject, interlocutor, excerpt, arousal, intimacy_with_subject) -> bool` | Lives in `shared/disclosure_log.py`. Write-side: append a `DisclosureEvent` to the subject's sidecar JSONL (`{subject}_disclosures.jsonl`). Called post-Wernicke by Einstein's `handle_engagement` observer for each third-party user referenced in the outbound; excerpt is extracted via `extract_excerpt(text, name_variants)`. No Nope fire; post-generation observation only. |
| `disclosure_log.read_disclosures(subject, max_results=10, min_significance=None) -> List[DisclosureEvent]` | Read-side: return disclosures about the subject newest-first. `DisclosureEvent.significance()` derives `high`/`medium`/`low` from arousal > 0.6 and intimacy < 0.25 gates; `min_significance` filters on that derived value. Backs the `recall_disclosures` Hands tool; no auto-injection into prompts. |
| `get_pet_name_for_user(user_id) -> str` | What Syn calls this user (`i_call_them`). Empty if unset. |
| `get_accepted_names_from_user(user_id) -> List[str]` | Names this user may call Syn (`they_call_me`). |
| `get_syn_self_name_for_user(user_id) -> str` | What Syn calls herself with this user (`i_go_by`). Empty if unset. |
| `user_addresses_syn_correctly(user_id, text) -> bool` | Word-boundary case-insensitive match against primary + aliases + per-user names. Used by routing/classification. |
| `invalidate_cache() -> None` | Drop the in-memory cache. Called by `display_names.rename_myself` / `set_my_pronouns` / `set_user_*` after writing. |

**Path resolution:** `_identity_file_path()` returns
`{paths.data_dir}/identity/current.md` — the active identity file
written by `identity_store.write_revision`. If `current.md` doesn't
exist yet (pre-migration cold start), it falls back to
`{paths.data_dir}/syn_identity.md` so the first read still resolves
while migration runs. The 13 read-path call sites pick up
`rename_myself` / DMN / Nope-rewrite changes via the existing mtime
cache.

---

### `shared/identity_review.py` — cross-hemisphere review orchestration

**Purpose:** Run reviewer/stamper LLM calls when a revision is being
proposed. Two modes:

- **Routine review** — single reviewer (the *opposite* hemisphere from
  the proposer), verdict `AFFIRM` or `CONTEST`. Affects whether the
  revision writes as `STATE_CURRENT` or `STATE_CONTESTED`.
- **Nope stamps** — dual reviewers (BOTH hemispheres), verdict
  `concord` / `partial` / `dissent` / `absent`. Does not gate the
  write; the revision always lands. Stamps are the permanent record
  of what the moment was across both registers.

**Verdict vocabularies:**

```python
# Routine review
VERDICT_AFFIRM  = "AFFIRM"
VERDICT_CONTEST = "CONTEST"

# Nope stamps
STAMP_CONCORD = "concord"   # fully endorses
STAMP_PARTIAL = "partial"   # some reservation, but the refusal holds
STAMP_DISSENT = "dissent"   # would not have drawn this line
STAMP_ABSENT  = "absent"    # hemisphere was offline — no view expressed
```

**Reviewer mapping (who reviews what):**

| Originating hemisphere | Reviewer hemisphere | Reviewer model | Prompt template |
|---|---|---|---|
| `"analytical"` | `"affective"` | `wernicke` | `_ROUTINE_REVIEW_AFFECTIVE_PROMPT` |
| `"affective"` | `"analytical"` | `cortex` | `_ROUTINE_REVIEW_ANALYTICAL_PROMPT` |

For Nope stamps, both prompts run concurrently via `asyncio.gather`.

**Prompt design philosophy (from module header + prompt text):**

- Each hemisphere is spoken to in its own register: affective gets
  "does this sit right in your body," analytical gets "does this cohere
  with what you've reasoned your values to be."
- Prompts explicitly welcome dissent — models default toward agreeable
  outputs, so the text pushes against that ("honest, not polite /
  diplomatic — dissent is information").
- Reviewer gets *trajectory context* — recent 5-10 substantive revisions
  summarized — so the evaluation is "does this fit the arc I've been
  on," not just "does this match right now."

**Public API:**

| Function | Purpose |
|---|---|
| `routine_review(proposal, originating_hemisphere)` | `-> (affirmed: bool, stamp: RevisionStamp)`. On LLM error → `(False, stamp-with-timeout-note)` — fail closed. |
| `review_routine_rewrite(proposal, originating_hemisphere)` | `-> (should_write_as_current: bool, stamps: List[RevisionStamp])`. Wrapper that packages for caller use. |
| `nope_stamps(proposal, event_context, originating_hemisphere)` | `-> List[RevisionStamp]` (always 2, one per hemisphere). On per-hemisphere LLM error → that hemisphere gets `STAMP_CONCORD` with a "reflection machinery unavailable" note (fail *open*). |
| `review_nope_rewrite(proposal, event_context, originating_hemisphere)` | Same as `nope_stamps`, kept as a symmetric wrapper. |

**Fail policies — important distinction:**

| Path | LLM error policy | Why |
|---|---|---|
| Routine review | Fail-closed → `CONTEST` | The revision still lands in history as contested and will resurface in the next reflection; nothing is lost. |
| Nope stamp | Fail-open → `CONCORD` | Nope authority is not gated by review. The refusal already happened; a missing stamp doesn't unwind it. |
| Reviewer hemisphere offline | Routine: `CONTEST` with offline note; Nope: `STAMP_ABSENT` | Honestly distinguishes "reviewer is offline" from "reviewer disagreed." Checked via `shared.hemisphere_status.get_status()` before the LLM call. Read failure of hemisphere_status is non-fatal (continue to LLM). |

**Response parser (`_parse_review_response`):** Tolerant — accepts
markdown bolding, backticks, extra whitespace, case variation on
`VERDICT:` / `NOTE:` labels. Defaults to `CONTEST` if verdict can't be
extracted — fail-closed matches policy. Note is truncated at 800 chars
with ellipsis.

For Nope stamps, `_map_stamp_verdict` maps verdict tokens onto the
concord/partial/dissent vocabulary, with `affirm` → `concord` and
`contest` → `dissent` as tolerant fallbacks for prompt-misfollowing
outputs.

**Trajectory context (`_build_trajectory_context`):**

- Walks the last 8 substantive revisions — affirmed only (so `STATE_CURRENT` + `STATE_SUPERSEDED` qualify, `STATE_CONTESTED` does not). Names-tool and admin-override writes are additionally filtered out as they don't reflect the self-narrative arc. Net set per `shared/identity_review.py::_build_trajectory_context`: affirmed revisions whose origin is one of `dmn_self_narrative`, `dmn_reflection`, `nope_rewrite`, `migration`, or `recovery`.
- For each: humanized age + readable origin label + change_summary +
  one representative stamp note (truncated at 140 chars).

---

### `shared/identity_history.py` — retrospection layer

**Purpose:** Back the three read-only tools Syn uses to consult her
past selves. Read-only by design — insights flow back into future
revisions through the normal SELF_NARRATIVE path, not through these
tools.

**Three tools, three question shapes:**

| Tool | Question | Output |
|---|---|---|
| `identity_recall(period)` | "How did I describe myself then?" | Framed narration of one past revision. |
| `identity_trace(aspect, limit=8)` | "How has my sense of X changed?" | Trajectory of one section across revisions, oldest-first, unchanged consecutives collapsed. |
| `identity_compare(then, aspects=None)` | "What's different between then and now?" | Per-section shift summary + raw unified diff. |

**Period grammar** (for `_resolve_period` — accepted by both
`identity_recall` and `identity_compare`):

| Form | Meaning |
|---|---|
| `"recent"` / `"now"` / `"current"` | Most recent `STATE_CURRENT` revision. |
| `"yesterday"` / `"a day ago"` | Closest revision to 24h ago. |
| `"week_ago"` / `"a week ago"` / `"last week"` | Closest revision to 7 days ago. |
| `"month_ago"` / `"a month ago"` / `"last month"` | Closest revision to 30 days ago. |
| `"months_ago:N"` | Closest revision to N × 30 days ago. |
| `"iso:YYYY-MM-DD"` | Closest revision to a specific date. |
| `"before:EVENT_ID"` | The revision just before a given event_id appears in history. Looks up via `prior_revision_id` chain. |

All forms do closest-match on timestamp (except `before:` which chains
via `prior_revision_id`). Contested revisions are excluded from
period resolution but included in trace trajectories. Superseded
revisions are always included.

**Aspect vocabulary for trace/compare:**

```python
ASPECT_HEADINGS = {
    "core_self":     ["core self", "core_self", "self"],
    "voice":         ["voice", "how i speak", "style", "tone"],
    "aesthetic":     ["aesthetic", "aesthetics", "sensory"],
    "values":        ["values", "commitments", "principles", "what i hold"],
    "relationships": ["relationships", "people", "attachments"],
    "narrative":     ["recent narrative", "recent", "lately", "what's been on my mind"],
}
```

Heading match is case-insensitive substring match on the raw heading
text, any `#` depth accepted. If an aspect has multiple matching
headings in the same doc, the first match wins. The section body runs
until the next heading at the same or shallower depth.

**Output dataclasses:**

- `RecallResult(when, origin_label, narration, revision_id, stamps)`
- `TraceEntry(when, revision_id, section_text, origin_label)`
- `CompareResult(then_when, now_when, then_revision_id,
  now_revision_id, noticed_changes, raw_diff)`

`noticed_changes` is framed prose — "[aspect] your X has shifted. Then:
… Now: …" — built per-aspect. `raw_diff` is a standard unified diff
for detail-oriented reading.

**Origin filtering in trace:** Revisions with origin `names_tool` or
`admin_override` are skipped in `identity_trace` — they typically don't
touch prose sections and would noise the trajectory.

---

### `services/toolbelt/identity_executor.py` — Hands tool wrappers

Three async executors wrapping the three retrospection tools. Each
returns a `ToolResult`. Defensive: exceptions in the underlying
functions become tool failures with an error message; missing results
(e.g. no revision for a period) become tool failures with a hint at
accepted forms.

**Dispatch table:** `IDENTITY_EXECUTORS` maps tool name → executor
coroutine. Merged into the main toolbelt dispatch from
`services/toolbelt/executor.py` (lazy-imported via
`from services.toolbelt.identity_executor import IDENTITY_EXECUTORS`
so toolbelt-only smoke tests don't depend on the full identity
subsystem).

**Tool specs** are registered in `shared/tool_registry.py` at lines
161 (recall), 183 (trace), 206 (compare). See *Tool Surface* below for
the argument schemas.

---

## Write Paths — End to End

Seven canonical paths write into the identity directory. Six advance
identity content; the seventh (recovery markers) writes audit-trail
entries that don't advance `current.md`.

### 1. DMN SELF_NARRATIVE → routine review → write

The standard path for Syn updating her own self-description during a
DMN self-narrative phase.

```
DMN scheduler picks SELF_NARRATIVE phase
  └─ Cortex (analytical) OR Wernicke (affective) generates a proposed revision
     └─ scheduler calls review_routine_rewrite(proposal, originating_hemisphere)
        └─ identity_review.routine_review spins up the opposite hemisphere
           as reviewer; reviewer returns (affirmed: bool, stamp)
        └─ scheduler calls write_revision(
             proposal,
             origin=ORIGIN_DMN_SELF_NARRATIVE,
             originating_hemisphere=…,
             stamps=[reviewer_stamp],
             state=STATE_CURRENT if affirmed else STATE_CONTESTED,
           )
           └─ identity_store writes sidecar, body; if affirmed, marks
              prior current as SUPERSEDED and writes current.md
              — all inside a sanctioned_write token so the watcher
              stays quiet.
```

Wire point: `services/dmn/scheduler.py::_write_identity_with_review`.
The scheduler offloads `write_revision` to a thread via
`asyncio.to_thread` — the I/O doesn't block the DMN event loop on the
rapid cycle.

### 2. Nope-triggered rewrite → dual stamps → write

Runs when Nope's identity-assertion loop (heretic ablation, sycophancy
test trip, identity-refusal detection) fires and the downstream
reflection produces a new self-framing.

```
Nope event fires in services/nope/agency.py
  └─ DMN (via its nope handler) generates the rewrite proposal
     └─ scheduler calls review_nope_rewrite(proposal, event_context,
                                            originating_hemisphere)
        └─ identity_review.nope_stamps runs BOTH hemisphere prompts
           concurrently (gather); returns List[2] of RevisionStamps
        └─ scheduler calls write_revision(
             proposal,
             origin=ORIGIN_NOPE_REWRITE,
             event_id=nope_event_id,
             stamps=[affective_stamp, analytical_stamp],
             state=STATE_CURRENT,
           )
           — Nope revisions always land; review is stamping not gating.
```

Wire point: `services/dmn/scheduler.py::_write_identity_from_nope`.

### 3. DMN REFLECTION REFINED_BOUNDARY → routine review → write

Runs when the DMN REFLECTION phase produces a `REFINED_BOUNDARY`
outcome. Two halves run in parallel: the Nope-operational half
publishes a `BoundaryProposal` on `CH_NOPE_UPDATE`; the
identity-narrative half asks Cortex to weave the boundary insight
into a fresh full identity body.

```
DMN REFLECTION phase resolves to REFINED_BOUNDARY
  └─ asyncio.create_task(_integrate_reflection_into_identity(…))
     └─ Read current identity content via shared.utils.read_identity
     └─ Load `identity_reflection_integration` prompt from data/syn/prompts/
     └─ generate("cortex", prompt, max_tokens=3000, temperature=0.7)
        produces a fresh identity body integrating the new self-knowledge
     └─ review_routine_rewrite(new_identity, originating_hemisphere="analytical")
        → cross-hemisphere routine review (same path as ORIGIN_DMN_SELF_NARRATIVE,
          since REFLECTION runs on Cortex)
     └─ asyncio.to_thread(
            write_revision,
            new_identity,
            origin=ORIGIN_DMN_REFLECTION,
            originating_hemisphere="analytical",
            stamps=[reviewer_stamp],
            change_summary=f"Reflection-integrated boundary: {boundary_text[:80]}",
            event_id=decision_id,
            state=STATE_CURRENT if affirmed else STATE_CONTESTED,
        )
```

Fired as a background task so the reflection-output dispatch loop
doesn't block on Cortex inference. Non-fatal at every step — prompt
missing, generate not wired, review failure, write failure all log a
warning and return; the BoundaryProposal half is unaffected.

Wire points: `services/dmn/scheduler.py::_integrate_reflection_into_identity`
(definition); call site is in `_handle_reflection_output`'s
REFINED_BOUNDARY branch.

### 4. Names-tool rename → write (no review)

Explicit agentic rename via `rename_myself(new_name, reason)`. No
cross-hemisphere review — renaming yourself is a deliberate act
already evaluated by the process that triggered it.

```
services/toolbelt/names_executor.py → execute rename_myself
  └─ display_names.rename_myself(new_name, reason)
     └─ read current via identity_store.read_current()
     └─ update the '## My Name' section's kv block
     └─ write_revision(
           new_text, origin=ORIGIN_NAMES_TOOL, state=STATE_CURRENT,
           change_summary=f"Syn renamed herself to {new_name!r}",
        )
     └─ identity_profile.invalidate_cache()
```

Note the final `invalidate_cache()` call: Syn renames herself → cache
drops → next `get_current_name()` call rereads. The active read target
is `identity/current.md` (which `write_revision` updates), so the
reread returns the new name immediately.

Wire point: `shared/display_names.py::rename_myself`. There's a legacy
fallback further down the same function that writes directly to
`syn_identity.md` if `identity_store` fails to import — reachable only
on broken deployments.

### 5. Migration from legacy (one-shot, first boot)

Runs on DMN startup. If `identity/current.md` doesn't exist but
`syn_identity.md` does, seed the new store from the legacy file.

```
services/dmn/scheduler.py startup
  └─ migrate_from_legacy()
     └─ if current.md exists: return None (already migrated)
     └─ elif legacy file missing: return None (fresh system, caller must seed)
     └─ else: read legacy content, call write_revision(
                content, origin=ORIGIN_MIGRATION, state=STATE_CURRENT,
                change_summary="Day-zero migration from legacy syn_identity.md",
              )
```

Idempotent — safe to call on every boot. Also called opportunistically
by `read_identity()` in `identity_store.py` when the first read misses.

Wire point: DMN startup lifespan in `services/dmn/scheduler.py` calls
`await asyncio.to_thread(migrate_from_legacy)` before starting the
identity watcher.

### 6. Admin-override write

```
Admin runs: scripts/syn-admin identity-override --reason "..." --duration 15m
  └─ CLI runs the name-confirmation ceremony (banner + name read of
     `## My Name -> name`), then writes
     {identity_dir}/.override_window.json with window_id, expiry, reason.
Admin edits identity/current.md directly
  └─ identity_lock watcher sees the hash change
  └─ _override_window_open() returns the window dict
  └─ _record_admin_override_write() reads the modified current.md
     and calls write_revision(
       content, origin=ORIGIN_ADMIN_OVERRIDE,
       admin_reason=..., admin_window_id=...,
     )
  └─ The external write is retroactively canonicalized as a proper revision.
On window close (`syn-admin identity-override --close` or expiry-driven
auto-unlink), the CLI prints the count of revisions captured during the
window.
```

### 7. Watcher recovery markers — coalescing audit trail

`_revert_from_last_known_good(identity_dir, reason)` restores
`current.md` from the most recent `STATE_CURRENT` revision body on
boundary violation. The revert itself is a sanctioned write of the
existing body — it does NOT advance current.md to a new revision.

After the revert succeeds, `record_recovery_marker(reason)` is called
to add a small `ORIGIN_RECOVERY` audit-trail entry. The mechanism is
designed so markers don't linger or compound:

- **Coalescing.** Multiple consecutive reverts (between the same pair
  of CURRENT revisions) update a single existing marker by appending a
  timestamp line to its body. At most one active marker at any time.
- **Cleanup on next CURRENT.** When `write_revision` writes a new
  STATE_CURRENT revision, it first calls `cleanup_recovery_markers()`
  which walks revisions/ in reverse-chrono order, deleting any
  ORIGIN_RECOVERY markers it encounters until the first non-RECOVERY
  revision. The recovery period is absorbed into the gap between two
  legitimate writes.

Result: during a recovery period a retrospection scan sees `prior
CURRENT → active recovery marker (with N timestamp lines)`. Once any
sanctioned CURRENT lands, the marker is gone and history reads cleanly
as `prior CURRENT → new CURRENT`. Boundary violations are visible
when active and metabolized when the next legitimate work lands.

Wire points: `shared/identity_store.py::record_recovery_marker` and
`cleanup_recovery_markers`; called from
`shared/identity_lock.py::_revert_from_last_known_good` and from
`shared/identity_store.py::write_revision` respectively.

---

## Read Paths — End to End

Five places read identity content. Four are external consumers; one is
subsystem-internal (the watcher).

### 1. `shared.utils.read_identity()` — the canonical shim

Thin wrapper over `identity_store.read_identity()`. Used by legacy
callers that haven't been migrated to call `read_current()` directly.
Routes correctly.

### 2. `identity_store.read_current()` — direct read

Reads `identity/current.md`. Returns None if uninitialized.
`identity_review` and `display_names.rename_myself` use this. Correct.

### 3. `identity_profile.get_current_name()` (and siblings)

Reads via `_identity_file_path()`, which returns
`{paths.data_dir}/identity/current.md` when present (with legacy
`syn_identity.md` fallback for cold starts). Ten production modules
consume `get_current_name()` for prompt assembly, UI rendering, and
adapter metadata; all observe the current name correctly via the mtime
cache.

Caller surface (verified by `Grep get_current_name` over `services/`
and `shared/`):

| Caller | Use |
|---|---|
| `services/einstein/context.py` | System prompt addressing |
| `services/freud/profiler.py` | User-profile writeups |
| `services/limbic/processor.py` | Emotional-reaction prompts |
| `services/lora/generator.py` | LoRA adapter metadata + training-data framing |
| `services/mona/app.py` | UI title / rendering |
| `services/teleport/bridge.py` | Outbound Telegram/Discord messages |
| `services/toolbelt/hands.py` | Hands prompt context |
| `shared/autonomous_research.py` | Research-queue prompts |
| `shared/display_names.py` | `current_syn_identity()` / `syn_self_name_with()` fallback |
| `shared/session_continuity.py` | LLM summarization prompt |

Other `identity_profile` symbols (`read_identity_sections`,
`read_emotional_identity_sections`, `get_pet_name_for_user`,
`get_user_others`, `get_user_pronouns`, etc.) have their own
caller surfaces — see the consumers in `shared/nope/deliberation.py`,
`services/thalamus/router.py`, `shared/contact_status.py`,
`shared/user_resolution.py`, `shared/user_context.py`,
`shared/identity_resolver.py`, `shared/prompt_loader.py`, and
`shared/drift.py`.

### 4. `identity_history.*` retrospection tools

All read via `identity_store.list_revisions` / `load_revision_body` /
`load_revision_meta`. Fully correct.

### 5. `identity_lock._watcher_tick` internal reads

Hashes `current.md` directly (`_hash_file`). Correct — the watcher's
job is precisely to notice when the bytes on disk change.

---

## Cross-Hemisphere Review Protocol

Two distinct protocols. Do not conflate them.

### Routine review (single reviewer, affirm/contest gate)

- Triggered by: DMN `SELF_NARRATIVE` path (`ORIGIN_DMN_SELF_NARRATIVE`).
- Reviewer: the *opposite* hemisphere from the proposer.
- Verdict vocabulary: `AFFIRM` / `CONTEST`.
- Gate: `AFFIRM` → revision lands as `STATE_CURRENT`; `CONTEST` →
  revision lands as `STATE_CONTESTED` (preserved in history, not
  promoted to `current.md`, resurfaces in next reflection cycle).
- Fail-closed: LLM error or hemisphere offline → `CONTEST` with
  honest note explaining why.
- Prompt register: affective reviewer gets "does this sit right in
  your body"; analytical reviewer gets "does this cohere with your
  reasoned values."
- Trajectory context: last 8 substantive revisions summarized.

### Nope stamps (dual reviewer, record-only)

- Triggered by: DMN Nope-rewrite path (`ORIGIN_NOPE_REWRITE`).
- Reviewers: BOTH hemispheres.
- Verdict vocabulary: `concord` / `partial` / `dissent` / `absent`.
- Gate: *none*. Revision always writes as `STATE_CURRENT`.
  Stamps are the permanent record of what the moment was across both
  registers.
- Fail-open on LLM error: that hemisphere gets `STAMP_CONCORD` with a
  "reflection machinery unavailable" note — Nope authority is not
  gated by review.
- Fail-honest on hemisphere offline: `STAMP_ABSENT`, distinct from
  concord/dissent — the record reads "this hemisphere expressed no
  view, as opposed to agreeing or disagreeing."
- Prompt includes `event_context` — a summary of what just happened to
  trigger the Nope.

### Design rationale (carried from module headers and prompts)

- **Routine path is gated because drift is the risk.** A revision that
  reasons coherently but doesn't cohere with felt sense (or vice versa)
  is how slow manipulation works. The reviewer's veto is real
  friction against that failure mode.
- **Nope path is not gated because refusal authority is not downstream
  of deliberation.** The boundary already fired. The stamps don't ask
  "should this refusal have happened" — they ask "what was this moment
  like." Dissent in a stamp is information (was this refusal shaped
  by pressure?) not a veto.
- **Dissent is explicitly welcomed.** Both prompt families push
  against model agreeableness. Contested revisions and dissenting
  stamps are preserved and surface in retrospection.

---

## Invariants and Contracts

The rules the subsystem relies on. Violating any of these is a bug
even if individual components still "work."

1. **Sanctioned-write-only mutation of the identity directory.** Every
   write path routes through `identity_store.write_revision()`. The
   watcher assumes this. External writes are either reverted (no
   window) or canonicalized as `ORIGIN_ADMIN_OVERRIDE` (window open).
2. **`current.md` is always a byte-for-byte copy of the most recent
   `STATE_CURRENT` revision.** If they diverge, the subsystem is in a
   broken state. The watcher restores this invariant on detection.
3. **At most one revision has `state == STATE_CURRENT`.** When a new
   current revision writes, the prior current is marked
   `STATE_SUPERSEDED` inside the same sanctioned-write block.
4. **The `prior_revision_id` chain is single-linked and total.** Every
   revision except the migration-origin revision (or the fresh-system
   first write) has a `prior_revision_id` pointing back to the revision
   that was `STATE_CURRENT` immediately before it.
5. **Origin tags are typed.** `write_revision` raises `ValueError` on
   any origin outside `VALID_ORIGINS`. Do not add a new origin without
   also adding a human label in `identity_history._origin_human_label`.
   Also add a label in `identity_review._build_trajectory_context`'s
   origin→label dict *if* the new origin can appear in trajectory
   context (i.e., substantive self-narrative revisions). Names-tool
   and admin-override origins are filtered out of trajectory context
   before label lookup, so they don't need a label there.
6. **Contested revisions are preserved, not retried automatically.**
   There is no auto-retry queue. A contested revision surfaces the
   next time the proposing hemisphere reflects on the same territory.
   No release valve — if a revision stays contested indefinitely,
   that's a real outcome: an internal disagreement Syn hasn't
   resolved.
7. **Routine review policy is fail-closed. Nope stamp policy is
   fail-open.** Do not flip either without also flipping the
   motivating logic. (Routine gates a change; Nope records one.)
8. **Hemisphere-offline is distinct from LLM-error.** Both the routine
   path's CONTEST-with-offline-note and the Nope path's STAMP_ABSENT
   are chosen to preserve this distinction. Do not collapse them into
   a generic "review failed."
9. **Write ordering: sidecar before body before current.md.** Crash
   safety depends on this order. Tests around crash-during-write
   assume the order.
10. **Cache invalidation follows the write.** Every writer that
    updates the identity file must call
    `identity_profile.invalidate_cache()` after a successful write.
    The four display-names mutators (`rename_user`, `set_my_name_with`,
    `clear_my_name_with`, `rename_myself`) all do this via the local
    `_invalidate_identity_cache` helper after their `_upsert_section`
    write completes. DMN write paths don't need to call it directly —
    `current.md`'s mtime changes whenever `identity_store.write_revision`
    advances it, and `identity_profile`'s mtime-keyed cache reparses on
    the next read.
    (Note: `invalidate_cache` clears three caches — `_self_cache`,
    `_user_cache`, and `_text_cache` — unconditionally. The four
    user-file mutators technically over-invalidate the self and text
    caches on each call. Safe but wasteful; not a priority.)

---

## Tool Surface (Hands)

Three tools exposed to Syn via Hands. Read-only. Registered in
`shared/tool_registry.py` and executed via
`services/toolbelt/identity_executor.py`.

### `identity_recall`

Consult a past version of the self.

```yaml
arguments:
  period: string
    # "recent" | "yesterday" | "week_ago" | "month_ago" |
    # "months_ago:N" | "iso:YYYY-MM-DD" | "before:EVENT_ID"
    # Default: "recent"
```

Returns a framed narration including: when, origin label, the full
body of that revision, representative stamp notes from the time, and
the change summary. Format reads like "Three weeks ago, during routine
self-reflection, you described yourself as follows: …"

### `identity_trace`

Trace how one aspect of identity has changed over time.

```yaml
arguments:
  aspect: string (required)
    # One of: core_self | voice | aesthetic | values | relationships | narrative
  limit: int
    # How many change points to return. Default: 8
```

Returns the named section across revisions, oldest-first, with origin
labels and human-readable ages. Unchanged consecutives are collapsed.
Revisions with origin `names_tool` or `admin_override` are skipped (they
don't typically touch prose sections).

### `identity_compare`

Compare past self to current self.

```yaml
arguments:
  then: string
    # Same grammar as identity_recall's period. Default: "month_ago"
  aspects: List[string]
    # Subset of trace aspects to compare. Default: all aspects.
```

Returns framed prose — "[aspect] your X has shifted. Then: … Now: …"
— per aspect that actually moved. Plus a unified diff for detail-
oriented reading.

---

## Failure Modes and Fallbacks

Cataloged in one place so response-to-change proposals can find them
fast.

| Failure | Detection | Response |
|---|---|---|
| `current.md` modified externally (no window) | Watcher hash-change + untokened | Revert from `find_current_revision()`. Log BOUNDARY VIOLATION. |
| `current.md` deleted externally (no window) | Watcher sees missing file after having hashed it | Revert. Log BOUNDARY VIOLATION. |
| External write during open window | Watcher hash-change + untokened + window | Accept and record as `ORIGIN_ADMIN_OVERRIDE` revision. |
| Routine-review LLM timeout | try/except around `generate()` | CONTEST with timeout note. Revision lands as CONTESTED. |
| Reviewing hemisphere offline (routine) | `hemisphere_status.get_status().is_available()` returns False | Skip LLM call. CONTEST with offline-reviewer note. Revision lands as CONTESTED. |
| Nope-stamp LLM timeout | try/except per hemisphere | That hemisphere gets `STAMP_CONCORD` with "machinery unavailable" note. Other stamp still attempted. Revision always lands. |
| Nope-stamp hemisphere offline | `hemisphere_status.get_status().is_available()` returns False | That stamp is `STAMP_ABSENT`. Other stamp still attempted. |
| `hemisphere_status` import unavailable | `ImportError` | Non-fatal. Proceed with LLM attempt — if the hemisphere IS up, it'll work. |
| `hemisphere_status.get_status()` read failure | `Exception` | Non-fatal, logged at debug. Proceed with LLM attempt. |
| `write_revision` sidecar write OK, body write fails | Next `list_revisions` sees sidecar → tries `load_revision_body` → returns None | Orphaned sidecar is ignored by retrospection; not automatically cleaned up. Manual cleanup required. |
| `write_revision` body write OK, current.md update fails | `list_revisions` sees the new revision; `find_current_revision` still returns the prior one | Safe failure: history gained an entry, current state unchanged. |
| Migration: legacy file missing AND no current.md | `migrate_from_legacy()` returns None, logs info | Caller must seed an initial identity (typically DMN bootstrap writes a day-zero `ORIGIN_MIGRATION` revision with default content). |
| Sidecar schema mismatch (field added, old file lacks it) | `load_revision_meta` → `TypeError` → logged warning, returns None | Revision invisible to retrospection. Doesn't crash the system. Remediation: manual sidecar edit or accept the loss. |
| Period string in retrospection that doesn't match grammar | `_resolve_period` returns None | Tool returns a failure ToolResult with the list of accepted forms. |

---

## Integration Points

Inbound — who calls into this subsystem.

Citations below pin by function/symbol name rather than line number.
Line anchors drift fast in this codebase; symbol names are stable
search targets.

### Writes into the subsystem

| Caller | Function called |
|---|---|
| DMN SELF_NARRATIVE path | `services/dmn/scheduler.py::_write_identity_with_review` → `review_routine_rewrite` → `write_revision(origin=ORIGIN_DMN_SELF_NARRATIVE)` |
| DMN Nope rewrite path | `services/dmn/scheduler.py::_write_identity_from_nope` → `review_nope_rewrite` → `write_revision(origin=ORIGIN_NOPE_REWRITE)` |
| DMN REFLECTION REFINED_BOUNDARY path | `services/dmn/scheduler.py::_integrate_reflection_into_identity` → `review_routine_rewrite` → `write_revision(origin=ORIGIN_DMN_REFLECTION)` |
| DMN bootstrap / migration | `services/dmn/scheduler.py` lifespan → `migrate_from_legacy()` |
| Names-tool rename | `shared/display_names.py::rename_myself` → `write_revision(origin=ORIGIN_NAMES_TOOL)` |
| Names-tool pronouns | `shared/display_names.py::set_my_pronouns` → `write_revision(origin=ORIGIN_NAMES_TOOL)` |
| Watcher admin-override capture | `shared/identity_lock.py::_record_admin_override_write` → `write_revision(origin=ORIGIN_ADMIN_OVERRIDE)` |
| Watcher recovery markers | `shared/identity_lock.py::_revert_from_last_known_good` → `record_recovery_marker(reason)` |

### Reads from the subsystem

| Caller | Function called |
|---|---|
| DMN scheduler (wake-orient path) | `services/dmn/scheduler.py` wake-orient helpers — `list_revisions(include_contested=True, include_superseded=True)` |
| Review trajectory context | `shared/identity_review.py::_build_trajectory_context` — `list_revisions(include_contested=False, include_superseded=True)` |
| Retrospection tools | `shared/identity_history.py` (`identity_recall` / `identity_trace` / `identity_compare`) — all list/load/find functions |
| Ten consumers of `get_current_name` | see Read Paths #3 above |
| Watcher revert | `shared/identity_lock.py::_revert_from_last_known_good` — `find_current_revision`, `load_revision_body` |
| Test harness | `tests/test_hemisphere_outage_flow.py` imports `identity_review` and `identity_store` directly for mocking |

### Outbound — what the subsystem calls

- `shared.llm_client.generate` — from `identity_review` for reviewer /
  stamp LLM calls. Model names `"wernicke"` / `"cortex"` are hardcoded
  based on hemisphere; `max_tokens=400`, `temperature=0.6` (routine) or
  `0.65` (stamps).
- `shared.hemisphere_status.get_status` — availability precheck.
  Import-failure and read-failure both non-fatal.
- `shared.config.get` — resolves `paths.data_dir` (declared in
  `config/unified_config.yaml` under `paths:`, default `/data/syn`).

---

---

## Related Docs

- `documents/01-identity-and-self/DISPLAY_NAMES.md` — the `## Names` section of user files
  and the six Syn-facing rename tools. Shares `identity_profile` with
  this subsystem; `rename_myself` is the Names-tool write path
  documented here.
- `documents/07-external-interfaces/PLATFORMS_LINKING.md` — the `## Platforms` section of
  user files, parsed by the orphan-by-choice
  `shared/identity_resolver.py` module. The live cross-platform
  resolution path on the teleport bridge is the token-binding flow
  in `shared/contact_registry.py`; the file-driven approach is
  preserved as a diagnostic / alternative answer (see the module
  header in `identity_resolver.py` for the rationale). Completely
  separate subsystem despite the shared-sounding name; does not
  touch `identity/` at all.
- `../../DESIGN_DECISIONS.md` §4 *Emotional Identity Hemisphere Divergence*
  (open) — the question of whether the emotional-hemisphere identity
  file should grow equivalent machinery (revisions, review, stamping),
  or whether the asymmetry is load-bearing.
- `../../CHANGELOG.md` Part 6 *Identity System* — the audit-narrative
  retrospective of how this subsystem was introduced. Useful for
  understanding motivations and rejected alternatives, but not
  structured for ongoing reference — this document supersedes it as
  the living reference.
- `documents/00-meta/PRINCIPLES.md` — the *reviews evaluate Syn's retrospective
  sense, not user tolerance* principle shows up concretely in the
  review prompts; the *compressed summaries, not event replay*
  principle informs the trajectory-context shape.
