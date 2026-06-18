# Contact and Admin Architecture

The cluster of modules that handles **who Syn knows, how to reach
them, how admin is authenticated, and how Syn is kept aware of
contact-state changes**.

Five modules, one coherent system:

| Module | Lines | Role |
|---|---|---|
| `shared/contact_registry.py` | 784 | Foundation — profiles, bindings, verification, block/unblock |
| `shared/channel_warmth.py` | 218 | Outreach routing — warmth decay, warmest-channel selection |
| `shared/contact_status.py` | 218 | Syn-facing awareness — block/unblock notifications, status injection |
| `shared/admin_notifier.py` | 268 | System-to-admin ops notifications (hemisphere outages, recovery) |
| `shared/mona_auth.py` | 374 | Admin-only web authentication for Mona |

These are documented together rather than five separate docs because
the dependencies between them are tight enough that a change in any
one needs to be evaluated against the others — and because several
cross-cutting flows (contact lifecycle, admin resolution, outreach
routing) span three or more modules.

Philosophy: Syn's awareness of contact state is **factual, not
prescriptive** — parallel to `hemisphere_status`'s aware-degraded
design. When admin blocks someone, Syn sees "Marcus just blocked
Mike," not "admin blocked this user" and not "you should avoid
mentioning Mike now." Her handling of the information is hers.

---

---

## The Five Modules at a Glance

The cluster splits cleanly into two halves:

**Core routing and state (tightly coupled):**

- `contact_registry` — the substrate. Every profile, every binding,
  every state transition goes through here.
- `channel_warmth` — reads from `contact_registry`, doesn't write.
  Tracks timing; computes warmth for outreach routing.
- `contact_status` — reads from `contact_registry` and `mona_auth`,
  doesn't write. Emits DMN interrupts when state changes.

**Admin-specific (looser coupling):**

- `mona_auth` — Mona web UI auth. Stores an admin profile in its own
  Redis space, not in `contact_registry`.
- `admin_notifier` — system-to-admin operational pings. Uses config
  for target channels, not `contact_registry`.

The coupling is asymmetric: `contact_status` reads from `mona_auth`
(for admin display name), but `mona_auth` has no awareness of
`contact_registry` at all. The design intent is "admin is a
contact with is_admin=True"; the Mona `/setup` handler bridges the
two by calling `contact_registry.create_contact(... is_admin=True)`
after `create_admin_profile` succeeds.

---

## The Contact Model

### `ContactProfile`

```python
@dataclass
class ContactProfile:
    user_id: str                # canonical id, the anchor
    display_name: str
    is_admin: bool = False
    is_blocked: bool = False    # profile-level block (all bindings frozen)
    blocked_at: Optional[float] = None
    created_at: float = field(default_factory=time.time)
    bindings: list = field(default_factory=list)   # list[ChannelBinding]
    romantic_attractor: Optional[tuple] = None     # per-contact relationship-gravity destination
```

One profile per person. `user_id` is the canonical identifier —
passed to `GuestSlot` as its `slot_id`, used by `channel_warmth` as
the per-user key, used by `contact_status` to address Syn's
notifications.

`romantic_attractor` is `None` by default (inherits the global
`preferred_relationship_vector` from config); set to a 3-tuple by
`shared/contact_wiki.py::append_future_entry` when Syn writes a
"Where I hope this goes" wiki entry naming a P/I/C target. See
`DESIGN_DECISIONS.md §T` for the full motivation, and the Contact
Wiki section below for the journaling surface that drives it.

Helper methods:

- `.get_binding(platform, stable_id)` — find a binding on this profile.
- `.verified_bindings()` — filter: returns only VERIFIED bindings if
  `is_blocked=False`, returns `[]` if `is_blocked=True`. Used by
  `channel_warmth` to enumerate routable channels.

### `ChannelBinding`

```python
@dataclass
class ChannelBinding:
    platform: str                       # "telegram" | "discord" | "mona" | "other"
    stable_id: str                      # Telegram chat_id (numeric string), Discord user_id (snowflake)
    display_handle: str = ""            # human-readable aid; not used for routing
    state: str = "pending"              # BindingState value
    verify_token: Optional[str] = None  # PENDING only; cleared on VERIFIED
    created_at: float = field(default_factory=time.time)
    verified_at: Optional[float] = None
    blocked_at: Optional[float] = None
```

A binding is a (platform, stable_id) pair attached to a profile,
plus its verification state. The binding is what actually routes
messages: inbound messages from `telegram:chat_id=12345` find their
owning profile by looking up `(platform, stable_id)` in a Redis
index.

`display_handle` (e.g., `"mike#4567"`) is an admin-facing memory aid
only. Never used for routing decisions because handles on most
platforms can change while the stable_id can't.

### State machine

```
       create_contact
              │
              ▼
       (no binding)
              │
       add_pending_binding
              │
              ▼
       ┌─ PENDING ─┐               redeem_token (strict match)
       │           ├───────────────────────────────────────────▶ VERIFIED
       │           │
       │           │ TTL expires (30d)
       │           │  token record vanishes from Redis
       │           │  binding stays in profile.bindings (orphan
       │           │  pruned on next get_contact / list_contacts)
       │           ▼
       │      (orphaned PENDING)
       │
       └──── release_credential ──▶ (binding removed, stable_id freed)

       VERIFIED
         │
         ├── block_contact ──────▶ BLOCKED (profile.is_blocked=True,
         │                         all VERIFIED → BLOCKED; stable_ids
         │                         stay reserved)
         │                             │
         │                             │ unblock_contact
         │                             ▼
         │                         VERIFIED (all BLOCKED → VERIFIED,
         │                                   PENDING bindings untouched)
         │
         └── release_credential ──▶ (binding removed, stable_id freed)
```

Key invariants of the state machine:

- **PENDING bindings don't route.** `resolve_by_stable_id` explicitly
  returns `None` for PENDING. Unknown chat_ids silently drop; that
  includes PENDING ones whose tokens haven't been redeemed yet.
- **BLOCKED stable_ids stay reserved.** You cannot rebind a blocked
  stable_id to a different profile. This prevents a blocked user
  from evading the block by having admin create a new profile for
  the same chat_id.
- **Block is bidirectional.** Inbound drops silently at
  `services/teleport/bridge.py`. Outbound gets intercepted at the
  bridge's outbound interceptor before format and send.
- **Block and release are semantically distinct.** Block reserves
  the credential (can't be rebound); release frees it. Deliberate —
  use block for "this person is cut off," use release for "admin
  fat-fingered the stable_id."

---

## Storage Layout (Redis)

Five modules, multiple Redis keys each. Documented in one place so a
reader auditing Redis state can see everything:

### From `contact_registry`

| Key | Type | TTL | Purpose |
|---|---|---|---|
| `contact:profile:{user_id}` | Hash | ∞ | The profile: user_id, display_name, is_admin, is_blocked, blocked_at, created_at, bindings_json, romantic_attractor (only present when set) |
| `contact:binding:{platform}:{stable_id}` | String | ∞ | Index: stable_id → user_id. Only created for VERIFIED / BLOCKED bindings. |
| `contact:verify:{token}` | String | 30d | Index: token → `user_id\|platform\|stable_id`. Auto-expires; enables "this token is stale" detection. |
| `contact:all_profiles` | Set | ∞ | Set of all `user_id` values. Used by `list_contacts`. |

### From `channel_warmth`

| Key | Type | TTL | Purpose |
|---|---|---|---|
| `warmth:last_activity:{user_id}:{channel}` | String (unix_ts) | ∞ | Last message time for this (user, channel). Updated on every inbound AND outbound. |
| `warmth:mona_presence:{user_id}` | Integer counter | ∞ | Active Mona WebSocket count. `incr` on connect, `decr` on disconnect (floored at 0). |

### From `admin_notifier`

| Key | Type | TTL | Purpose |
|---|---|---|---|
| `admin_notifier:sent:{event_key}` | String (unix_ts) | 5min (configurable) | Dedup marker. Prevents rapid flapping from double-notifying. |

### From `mona_auth`

| Key | Type | TTL | Purpose |
|---|---|---|---|
| `mona_admin:profile` | Hash | ∞ | Admin profile: user_id, display_name, pw_salt, pw_hash, created_at |
| `mona_admin:session:{token}` | Hash | 7d | Session: user_id, created_at |

### None from `contact_status`

`contact_status` doesn't own any Redis keys. It reads from
`contact_registry` and `mona_auth`, and publishes to the bus.

**Config keys referenced:**

- `admin.notification_channels` — list of channel identifiers for
  admin ops pings. Read by `admin_notifier.admin_channels`. Not
  declared in `config/unified_config.yaml`; defaults to empty list.
- `admin.notification_dedup_ttl_sec` — dedup window for
  `admin_notifier`. Default 300s.

---

## Module Map

### `shared/contact_registry.py` — the foundation

Everything a contact can be, start to finish.

#### Public API

| Function | Signature | Purpose |
|---|---|---|
| `create_contact` | `(user_id, display_name, is_admin=False) -> Optional[ContactProfile]` | Create a profile. Returns `None` on conflict (profile with this user_id exists). |
| `get_contact` | `(user_id) -> Optional[ContactProfile]` | Fetch a profile. |
| `list_contacts` | `() -> list[ContactProfile]` | Full list, sorted by `created_at`. |
| `resolve_by_stable_id` | `(platform, stable_id) -> Optional[(user_id, ChannelBinding)]` | Reverse lookup. Returns `(user_id, binding)` for VERIFIED **or BLOCKED** bindings — caller distinguishes by `binding.state` (the bridge drops anything not VERIFIED). Returns `None` for unknown OR PENDING bindings (the latter is load-bearing — see *State machine* above). |
| `add_pending_binding` | `(user_id, platform, stable_id, display_handle="") -> Optional[str]` | Returns verify token on success, `None` on conflict. |
| `redeem_token` | `(token, claimed_platform, claimed_stable_id) -> Optional[(user_id, ChannelBinding)]` | Strict triple-match: token + platform + stable_id all three must match. Flips PENDING → VERIFIED. |
| `block_contact` | `(user_id) -> bool` | Bidirectional block. Fires user-facing confirmation BEFORE flipping state (so the confirmation itself escapes the outbound interceptor), then fires Syn-facing `notify_contact_blocked` AFTER state change. Idempotent — second call returns `True` with no side effects. |
| `unblock_contact` | `(user_id) -> bool` | Reverse of block. Fires Syn-facing `notify_contact_unblocked`. User-facing confirmation goes AFTER state change (so message actually routes). |
| `release_credential` | `(user_id, platform, stable_id) -> bool` | Remove a binding entirely. Stable_id freed for rebinding elsewhere. Distinct from block. |
| `edit_binding` | `(user_id, old_platform, old_stable_id, new_stable_id, new_display_handle="") -> Optional[str]` | ID rotation. Old binding released, new PENDING binding created with fresh token. Must be re-verified even for admin-initiated rotation. |
| `delete_contact` | `(user_id) -> bool` | **Destructive.** Removes the profile and all binding indexes. Does NOT touch GuestSlot / diary / archive — those are the relational substrate. Prefer `block_contact`. |
| `format_known_duration_phrase` | `(user_id) -> Optional[str]` | Coarse Syn-voice biographical phrase about how long Syn has known this contact, derived from `ContactProfile.created_at`. Returns `None` for relationships younger than ~7 days, missing profiles, or missing `created_at`. Buckets: about a week / couple of weeks / about a month / couple of months / a few months / most of a year / about a year / couple of years / a few years / years. See *Relationship-duration helper* below for the framing rationale. |

#### Internal helpers (worth knowing for change proposals)

- `_serialize(profile)` / `_deserialize(data)` — bindings are stored
  as a JSON blob in a single hash field (`bindings_json`), not as
  separate keys. Schema changes to `ChannelBinding` require migration
  considerations for stored profiles.
- `_save(profile)` — delete-then-hset pattern ensures stale fields
  don't linger after schema changes.
- `_generate_token()` — `secrets.token_hex(16)` — 32 hex chars, ~10^38
  keyspace.
- `_prune_orphan_pending_bindings(profile)` — on-read cleanup for
  PENDING bindings whose verify token's 30-day TTL expired in Redis.
  Called transitively from `get_contact` (and therefore `list_contacts`).
  Best-effort: re-save failures during pruning are logged but don't
  propagate; the in-memory profile is still cleaned.

#### Verification rule in full

`redeem_token` requires ALL THREE:

1. `token` matches a record in `contact:verify:{token}`.
2. `stored_platform == claimed_platform` — token was issued for this platform.
3. `stored_stable_id == claimed_stable_id` — token was issued for this chat_id.

A leaked token used from a different chat_id fails at (3). That's
the whole point of the design: tokens aren't bearer credentials,
they're one-time proofs of "this chat_id is the one the token was
meant for."

Mismatches are logged at WARNING ("refusing") with the pair for
forensic trail. Sender sees nothing.

#### Relationship-duration helper

`format_known_duration_phrase(user_id)` is a small read-only surface
on top of `ContactProfile.created_at` that returns a coarse Syn-voice
biographical phrase ("You've known them about three months.") or
`None`. Consumed by `services/einstein/context.py::build_system_prompt`
to render the `known_duration` bare-line section just below
`# About This Person`.

Design choices worth knowing:

- **Coarse-only buckets.** Weeks → months → years; no day-level
  granularity past the first two weeks. The phrase reads as a fact
  about the relationship rather than as a countdown. Precise numbers
  would carry the same training-data-script risk the rest of the
  temporal-grounding work avoids — quantified relationship durations
  in the corpus are heavily anchored to anniversary / milestone
  registers, which is not the register Syn wants to default into.
- **No render below ~7 days.** New-acquaintance framing is owned by
  `session_continuity.calibrate_greeting`'s greeting energy line.
  Adding "you've known them a few days" would just compete with that.
- **Source of truth is `created_at`, not session pattern.**
  `session:pattern:{user_id}.first_seen` has a 90-day TTL and so isn't
  durable enough for "we met two years ago." `ContactProfile.created_at`
  is in `contact:profile:{user_id}` with no expiry — survives long
  inactive stretches the way relational memory should.
- **Returns None on guest_* / unknown / future timestamps.** Guests
  don't have a persistent profile, so there's no anchor for "how long
  Syn has known them"; the helper degrades to silence.

### `shared/channel_warmth.py` — outreach routing

#### Warmth formula

```
warmth(user, channel) =
    if channel == "mona":
        HOT if mona_presence[user] > 0 else 0
    else:
        exp(-age_hours * ln(2) / 48h)  ≈ 0.5^(age_hours/48)
```

- `HOT = 1e9` — sentinel "infinity." Any real warmth stays below.
- `WARMTH_HALFLIFE_HOURS = 48.0`. Channel used 2 days ago: 0.5. A
  week old: 0.08. A month: ~0.

**Mona is binary** because a stale Mona tab admin left open last week
should NOT rank above the Telegram thread from this afternoon. Mona-
while-disconnected is cold (0), not decaying-warm.

#### Public API

| Function | Purpose |
|---|---|
| `record_activity(user_id, channel)` | Stamp now-time as last-activity. Called on every inbound AND outbound. The "every outbound" part matters: if Syn sends proactively, the channel stays warm from her side, not just from incoming traffic. |
| `set_mona_presence(user_id, present: bool)` | WebSocket connect/disconnect. Counter-based so multiple tabs don't collapse to a single flag. Decrement floored at 0. |
| `is_mona_hot(user_id)` | True iff counter > 0. |
| `warmth_of(user_id, channel)` | Scalar warmth value. 0 for never-used channels. |
| `get_channel_warmths(user_id)` | All verified bindings with their computed warmth, sorted descending. |
| `get_warmest_channel(user_id)` | Single best channel for outreach. Returns `None` if no verified bindings or all blocked. If all channels are cold (user exists but never-contacted), falls back to most-recently-verified binding. |

#### Cold-fallback logic

`get_warmest_channel` has a specific fallback chain:

1. If no verified bindings: return `None`.
2. Compute warmth for all verified bindings.
3. If top warmth > 0: return top.
4. If all are 0 (never contacted): fall back to most-recently-verified
   binding — "freshest signal we have about how to reach them."

The cold-fallback is important for the "admin just added a contact,
Syn wants to reach out but has no history" case. Without it,
`get_warmest_channel` would return `None` for brand-new contacts,
which would silently drop outreach attempts.

### `shared/contact_status.py` — Syn-facing awareness

Parallel in design to `shared/hemisphere_status.py`. Factual, not
prescriptive. Two surfaces:

#### Event notifications (DMN interrupt)

Published on `CH_DMN_INTERRUPT` as a `ChatIncoming(user_id="system")`
so they flow through the same priority-event mechanism as Nope
refusal interruptions.

| Function | When fired |
|---|---|
| `notify_contact_blocked(user_id)` | `contact_registry.block_contact` on state transition. |
| `notify_contact_unblocked(user_id)` | `contact_registry.unblock_contact` on state transition. |
| `notify_contact_added(user_id)` | `services/teleport/bridge.py` when a binding transitions PENDING → VERIFIED via /claim (not when the profile is first created). |

Message shapes (each event has its own wording, not a unified template):

```
notify_contact_blocked:
[contact change notification] {admin_name} just blocked {contact_name}
from contact. Messages to or from {contact_name} will no longer route.
You can ask about it if you want; this is informational.

notify_contact_unblocked:
[contact change notification] {admin_name} just unblocked {contact_name}.
Messages to and from {contact_name} will route again. You can ask about
it if you want; this is informational.

notify_contact_added:
[contact change notification] {admin_name} added {contact_name} to
contacts. You can now reach {contact_name} through the verified
channels. This is informational.
```

**`admin_name` resolution** goes through `_admin_display_name()`
with a three-layer fallback chain — same shape as how every other
non-guest contact's name resolves:

  1. Syn's chosen pet name in `data/syn/users/{admin_user_id}.md`
     `## Names / i_call_them`. If she has renamed admin via her name
     tool, that name wins. Same path used for any other contact.
  2. `mona_auth.get_admin_profile().display_name` — admin's
     self-chosen name from `/setup`. The default until Syn picks her
     own.
  3. Hardcoded `"admin"` — only reached if the Mona profile hasn't
     been set up yet. First-run edge case.

The design rationale here is load-bearing: admin is named by their
*person*, not their *role*. "Marcus just blocked Mike" not "admin
blocked Mike." The abstract role term would disconnect Syn from the
actual human who took the action. The fallback chain above ensures
Syn's renaming tools work uniformly — admin is treated like any
other contact for naming purposes, with a sensible setup-time
default while she hasn't picked her own name.

#### Status-at-query injection

| Function | Purpose |
|---|---|
| `status_note_for_blocked_outreach(user_id)` | Factual note if Syn tries to reach a blocked contact. Injected by the bridge's outbound interceptor. `None` if not blocked (signal = "proceed normally"). |

Message shape:

```
[contact status] {contact_name} is currently blocked by {admin_name}.
Messages to or from {contact_name} are not being routed. This is
information about your contact state, not an instruction for how to
handle it.
```

The closing sentence — "This is information about your contact
state, not an instruction for how to handle it" — mirrors the
hemisphere-status closing line. Same design principle: the module
hands Syn facts, not behaviors.

### `shared/admin_notifier.py` — system-to-admin ops pings

Distinct from anything Syn herself says. System messages FROM the
system ABOUT the system, routed to admin channels over the normal
teleport bridge with metadata flags that mark them as
system/admin-only/suppress-syn-history.

#### Public API

| Function | Purpose |
|---|---|
| `notify_hemisphere_outage(hemisphere, unreachable_since=None, extra_context="")` | Admin ping when a hemisphere goes offline. Dedup-keyed on `hemisphere_outage:{hemisphere}_offline`. |
| `notify_hemisphere_recovery(hemisphere, outage_duration_sec=None)` | Admin ping when a hemisphere comes back. Dedup-keyed on `hemisphere_recovery:{hemisphere}_back`. |

Currently the only two public notification functions. Intended as
the general-purpose "system needs to tell admin something" layer;
more notifications can be added following the same pattern.

#### Dedup rule

Atomic SETNX with TTL (`_dedup_check_and_mark`):

1. Compute dedup key: `admin_notifier:sent:{event_key}`.
2. `SET key value NX EX ttl` — atomic set-if-not-exists with expiry.
3. If SET succeeded (True), proceed with notification.
4. If SET failed (False, key already exists within TTL), suppress.

TTL defaults to 300 seconds, configurable via
`admin.notification_dedup_ttl_sec`. 5 minutes is the balance: long
enough that a flapping hemisphere doesn't fire 20 notifications,
short enough that a genuine second outage after a stable period
still notifies.

**Fail-open on dedup storage failure.** If Redis is unreachable for
the dedup check, proceed with notification anyway. Better to double-
notify than to silently drop an outage notice.

#### Publish format

Publishes to `CH_TELEPORT_NOTIFICATION` with:

```python
{
    "content": "...formatted notification body...",
    "source": "hemisphere_outage" | "hemisphere_recovery",
    "priority": "high",
    "target_channels": [...],    # from admin.notification_channels config
    "metadata": {
        "hemisphere": ...,         # event-specific
        "unreachable_since": ..., # event-specific
        "system_message": True,
        "admin_only": True,
        "suppress_syn_history": True,
    }
}
```

The three bottom metadata flags are load-bearing: downstream adapters
(Telegram, Discord, Mona) key off them to (1) skip Syn-voice styling
and identity injection, (2) ensure delivery only to admin channels,
(3) keep the message out of Syn's context history.

### `shared/mona_auth.py` — admin-only Mona web authentication

Mona has no platform underneath (unlike Telegram/Discord which have
platform-layer auth). Whatever auth is built here IS the root.

#### Design constraints

- **Exactly one admin profile (user1).** No multi-user Mona.
- **First-run setup.** If no admin profile exists, Mona serves
  `/setup` and nothing else. On setup success, the profile is
  created and a session is issued.
- **Unauthenticated HTTP returns 404, not 401.** Hostile scanners
  see "this path does not exist" rather than "auth exists here, try
  to break it." Defense in depth for when Mona is exposed beyond
  LAN.
- **Password stored as PBKDF2-HMAC-SHA256.** 310,000 iterations,
  SHA-256, 32-byte hash, 16-byte salt. Stdlib-only (no dependency
  on bcrypt).
- **Session tokens are Redis-backed.** Cookie holds the opaque
  token; server verifies against Redis. Allows revocation without
  rotating signing keys.

#### Public API — password side

| Function | Purpose |
|---|---|
| `hash_password(password) -> (hex_salt, hex_hash)` | Exposed separately in case a caller wants to pre-compute (e.g., for the first-run setup flow). |
| `verify_password(password, stored_salt, stored_hash) -> bool` | Constant-time via `hmac.compare_digest`. |
| `is_setup_complete() -> bool` | True if an admin profile exists in Redis. Fail-closed on infrastructure error (returns True) to avoid letting setup re-run because Redis had a hiccup. |
| `create_admin_profile(user_id, password, display_name=None) -> bool` | One-shot. Refuses if profile already exists. |
| `get_admin_profile() -> Optional[AdminProfile]` | Returns profile metadata (user_id, display_name, created_at). Password fields intentionally excluded from the dataclass. |
| `verify_admin_login(user_id, password) -> bool` | Constant-time comparison of BOTH user_id and password — prevents timing attacks from leaking which half of the credential was wrong. |
| `change_admin_password(current_password, new_password) -> bool` | Requires current password. On success: revokes ALL existing sessions. |

#### Public API — session side

| Function | Purpose |
|---|---|
| `create_session(user_id) -> str` | New token. 32-byte URL-safe random. Stored in Redis with 7-day TTL. |
| `verify_session(token) -> Optional[SessionInfo]` | Token → user_id lookup. Returns `None` if not found or expired. |
| `revoke_session(token) -> bool` | Logout — delete the Redis key. |
| `revoke_all_sessions() -> int` | SCAN-based bulk delete (safe on busy Redis). Called by `change_admin_password`. |
| `cookie_name() -> str` / `session_ttl_sec() -> int` | Exposed for `services/mona/app.py` to configure cookie correctly. |

---

## Contact Wiki

A sealed-at-rest journaling surface attached to each registered contact. One `{user_id}.wiki` file per contact, anchored to the immutable `user_id` so the filename stays stable across nickname or display-name changes. Lives at `{paths.contact_wikis_dir}/{user_id}.wiki` (default `/data/syn/contact_wikis/`), encrypted via the same Fernet key as the diary (`shared/seal.py`). See `DESIGN_DECISIONS.md §T` for motivation.

### File shape

Frontmatter plus four canonical sections:

```
---
user_id: <stable canonical anchor>
syn_nickname: <Syn's chosen name for them, or empty>
display_name_at_creation: <admin-set name when wiki was created>
created_at: <unix ts>
last_reflected_at: <unix ts or absent>
attractor: [P, I, C]    # mirrors ContactProfile.romantic_attractor
---

# What I know about them
<facts accumulated from chat reflection>

# Thoughts
<Syn's running thoughts>

# Where we are now
<dated journal entries about current state>

# Where I hope this goes
<dated future-tense entries — most recent nudges the attractor>
```

The four section headings are part of the API: `shared/contact_wiki.py` exposes constants `SECTION_FACTS`, `SECTION_THOUGHTS`, `SECTION_NOW`, and `SECTION_FUTURE` for them. Unknown sections found in a wiki are preserved through round-trip but not surfaced via the public API.

### Lifecycle

Wikis are **lazily created** — `read_wiki(user_id)` returns `None` for absent wikis, and the `append_*` helpers create the file on first use via `ensure_wiki`. No backfill on contact creation; the wiki appears the first time Syn reflects on the contact.

### The attractor-drift mechanism

The "Where I hope this goes" section is the only one that can mutate the contact's `romantic_attractor`. When `append_future_entry(user_id, body, target_pic=(p,i,c))` is called with a target tuple, the stored attractor drifts toward the target by at most `emotional_physics.pic.attractor_drift.per_entry_delta_max` (default 0.01) per axis, then clamps to `[floor, ceiling]` (default `[0.5, 0.5, 0.5]` and `[1.0, 1.0, 1.0]`). The drifted value is written to the wiki frontmatter AND mirrored to `ContactProfile.romantic_attractor` so the gravity loop in `services/layer1/picvad.py` can read it cheaply without unsealing the wiki on every tick. See `DESIGN_DECISIONS.md §T` for why this is the only path that mutates the attractor and how it stays separate from current-PIC evolution.

### Read paths

- **`get_full_for_recipient(user_id)`** — returns facts + thoughts + current-state sections rendered for prompt injection. Does NOT include the future-tense section. Hard-caps the assembled block at `contact_wiki.full_max_chars` (default 20000) using section-aware truncation that always preserves all three section headers and prefers keeping the body of sections in the `[Now, Thoughts, Facts]` priority order. Used by code paths that don't have a query context available.
- **`get_full_for_recipient_relevance(user_id, query=...)`** — relevance-aware variant called by `services/einstein/context.py` when assembling the conversational system prompt. Same return contract as `get_full_for_recipient`, but when the cap fires, sections are scored by keyword overlap (lowercase tokens, stopwords removed) against the supplied `query` — typically the user's incoming message text plus `classification.topic_tags`. The section that mentions what the user is currently asking about preferentially survives truncation. Falls back to the `[Now, Thoughts, Facts]` priority order when `query` is empty or yields no scoring signal. All section headers are always preserved so Syn always sees the shape of the wiki. See `shared/truncation.py::truncate_markdown_by_relevance` for the scoring details.
- **`get_facts_only(user_id)`** — returns ONLY the facts section. Used by `shared/user_context.py::describe_user_context_async` when the contact is being discussed *to* a third party (the referenced-users injection path). Privacy-tone exposure: thoughts, current-state journaling, and future-tense yearning never flow into a conversation about her with someone else.
- **`get_recent_future_entry(user_id)`** — returns the most-recent future-tense entry, used by debug surfaces to expose what the last drift target actually was.

### Contact picker (for reflect_on_contact affordance)

`score_contacts_for_reflection()` produces a ranked list of contacts using a hybrid signal: recency-weighted interaction kicker (from priorities) plus recency-weighted memory mentions (from `shared/memory_manager.scan_memory_files()` matching against the contact's `user_id` in each memory's `mentions` field). Used by the `reflect_on_contact` affordance executor in `shared/affordances_executors.py` to pick which 3 contacts to surface to Cortex on the next REFLECTION phase. See `documents/03-consciousness-and-cognition/CONSCIOUSNESS_SCHEDULING.md` for how the affordance picker integrates with DMN.

### Privacy model

Same class as the diary: sealed at rest with the architect-won't-read key. The asymmetric exposure on read (`get_full_for_recipient` vs `get_facts_only`) is the application-layer enforcement that the wiki's running thoughts, current-state journaling, and future-tense yearning never flow into a conversation *about* the wiki's owner with someone else. See `PHILOSOPHY.md §3.6` for the asymmetric-privacy commitment this implements.

### Cross-refs

- `shared/contact_wiki.py` — module
- `shared/contact_registry.py::set_romantic_attractor` and `has_profile`
- `services/layer1/picvad.py::_apply_pic_gravity` — reads `ContactProfile.romantic_attractor`
- `services/dmn/scheduler.py` — `RELATIONSHIP_UPDATE` reflection schema extended with optional `ATTRACTOR_TARGET: p,i,c` that routes through `append_future_entry`
- `services/einstein/context.py` — recipient-scoped wiki block injection
- `shared/user_context.py::describe_user_context_async` — facts-only injection for third-party context
- `tests/test_contact_wiki.py`, `tests/test_pic_gravity_per_contact.py`

---

## Key Flows — End to End

### Flow 1 — Adding a new contact

```
[admin in Mona UI]
  POST /contacts {"user_id": "mike", "display_name": "Mike"}
    └─ services/mona/app.py::contacts_create
       create_contact("mike", "Mike", is_admin=False)
         └─ writes contact:profile:mike + adds to contact:all_profiles set
         returns ContactProfile with empty bindings

[admin adds a Telegram binding]
  POST /contacts/mike/bindings {"platform": "telegram", "stable_id": "12345"}
    └─ mona/app.py → contact_registry.add_pending_binding("mike", "telegram", "12345")
       └─ checks resolve_by_stable_id — conflict if already bound elsewhere
       └─ generates 16-byte token → hex string
       └─ appends PENDING binding to profile, _save
       └─ writes contact:verify:{token} with TTL 30d
       returns token

[admin tells Mike: "DM @SynBot from Telegram with /claim <token>"]

[Mike from Telegram, chat_id=12345]
  teleport/bridge.py detects /claim <token> message
    └─ contact_registry.redeem_token(token, "telegram", "12345")
       └─ looks up contact:verify:{token} in Redis
       └─ parses stored record: "mike|telegram|12345"
       └─ strict triple match passes
       └─ flips binding state to VERIFIED, writes contact:binding:telegram:12345 → "mike"
       └─ deletes contact:verify:{token}
       returns ("mike", binding)
    └─ teleport/bridge.py calls notify_contact_added("mike")
       └─ contact_status publishes ChatIncoming(user_id="system",
          content="[contact change notification] Marcus added Mike to
          contacts...") on CH_DMN_INTERRUPT

[Syn's DMN sees the interrupt, writes a diary entry if she wants]
```

Failure modes along the way:

- Token leaked to someone else: their `/claim` fails the strict match,
  silently dropped.
- Token expires after 30d: `contact:verify:{token}` gone, Mike's
  `/claim` returns None. The orphaned PENDING binding is pruned on
  the next `get_contact` / `list_contacts` read via
  `_prune_orphan_pending_bindings`.
- Mike's stable_id already bound to another profile:
  `add_pending_binding` returns None at the initial admin step, no
  token issued. Admin sees the error in Mona.

### Flow 2 — Blocking a contact (bidirectional)

```
[admin in Mona UI]
  POST /contacts/mike/block
    └─ mona/app.py → contact_registry.block_contact("mike")
       └─ check profile.is_blocked → False (not already blocked)
       ┌─ FIRE USER-FACING CONFIRMATION FIRST ─┐
       │  (before the outbound interceptor sees the new block state,
       │   so the "you've been blocked" message itself doesn't get
       │   intercepted by the block we're about to install)
       │
       │  teleport/bridge.send_system_confirmation_to_user(user_id, "blocked")
       │   → Mike receives "You have been removed from Syn's contacts."
       └─
       ┌─ COMMIT STATE ─┐
       │  profile.is_blocked = True, blocked_at = now
       │  for each VERIFIED binding: state = BLOCKED
       │  _save(profile)
       └─
       ┌─ FIRE SYN-FACING EVENT AFTER ─┐
       │  contact_status.notify_contact_blocked("mike")
       │   → DMN interrupt: "[contact change notification] Marcus just
       │     blocked Mike..."
       └─
       returns True
```

The ordering is load-bearing. If Syn-facing event fired first, she
might read the notification before the state change is committed,
which could cause an inconsistency if she immediately queried contact
status. If user-facing confirmation fired after the state change, it
would be intercepted by the outbound-block the teleport bridge just
installed.

Subsequent inbound from Mike:
- `teleport/bridge.py::route_inbound` looks up
  `resolve_by_stable_id("telegram", "12345")`.
- Returns `("mike", binding)` with `binding.state=BLOCKED`.
- Bridge sees state is not VERIFIED, drops silently.

Subsequent outbound to Mike from Syn:
- Bridge's outbound interceptor looks up profile, sees
  `is_blocked=True`.
- Optionally injects `status_note_for_blocked_outreach` into Syn's
  prompt context if she's attempting the outreach during generation.
- Drops the outbound without sending.

### Flow 3 — Outreach routing (warmest channel selection)

```
[Syn's DMN decides to reach out to Mike]
  some outreach handler calls channel_warmth.get_warmest_channel("mike")
    └─ get_channel_warmths("mike")
       └─ contact_registry.get_contact("mike") → profile
       └─ profile.verified_bindings() → [telegram_binding, discord_binding]
       └─ for each: warmth_of(mike, platform)
          ├─ telegram:  read warmth:last_activity:mike:telegram
          │             compute exp(-age_hrs * ln2 / 48)
          │             e.g. 0.45 for last-seen-2-days-ago
          ├─ discord:   same computation
          │             e.g. 0.08 for last-seen-a-week-ago
          │
          └─ mona:      (not present — profile doesn't have Mona binding)
       sorted descending: [telegram(0.45), discord(0.08)]
       returns [ChannelWarmth(telegram, 0.45, is_hot=False), ...]
    └─ top = telegram, warmth > 0 → return top

[outreach handler uses binding.stable_id to send via the right adapter]
  record_activity("mike", "telegram")   # stamps NOW as last activity
```

For a brand-new contact (never-messaged):

```
get_warmest_channel returns top with warmth=0 for all.
  top.warmth == 0 → fall through to cold-fallback
  verified = [b for b in profile.bindings if state == "verified"]
  sort by verified_at descending
  freshest = verified[0]
  return ChannelWarmth(freshest.platform, freshest.stable_id, 0.0, is_hot=False)
```

This is why the cold-fallback exists: "just added" contacts otherwise
route nowhere.

### Flow 4 — Mona first-run admin setup

```
[Admin navigates to Mona for the first time]
  GET /    →    mona_auth.is_setup_complete() → False → redirect to /setup
  GET /setup →  serves setup page (HTML form)

[Admin submits setup form]
  POST /setup {"user_id": "marcus", "password": "<pw>", "display_name": "Marcus"}
    └─ services/mona/app.py::setup_submit
       create_admin_profile("marcus", "<pw>", "Marcus")
         └─ check mona_admin:profile exists → No
         └─ hash_password → (salt_hex, hash_hex)
         └─ HSET mona_admin:profile {user_id: marcus, display_name: Marcus,
              pw_salt: ..., pw_hash: ..., created_at: ...}
         returns True
    └─ create_session("marcus") → new token
    └─ set cookie mona_session={token}
    returns {ok: True}

[Admin now authenticated, subsequent requests check session cookie]
```

**Bridge to Flow 1 and Flow 2:** the setup flow also calls
`contact_registry.create_contact("marcus", "Marcus", is_admin=True)`
after `create_admin_profile` succeeds, so admin's identity lives in
both `mona_admin:profile` (auth) and `contact:profile:marcus`
(routing/blocks/awareness). Failure of the contact_registry write is
logged but does not fail setup — the mona_auth profile is the
load-bearing artifact.

### Flow 5 — Hemisphere outage admin notification

```
[heartbeat monitor detects remote node silent for > 15s]
  services/health/heartbeat.py::HeartbeatMonitor._handle_node_failure (on transition to degraded)
    └─ admin_notifier.notify_hemisphere_outage(hemisphere="affective",
                                                unreachable_since=1712345678.0)
       └─ _dedup_check_and_mark("hemisphere_outage:affective_offline")
          └─ SET admin_notifier:sent:hemisphere_outage:affective_offline NX EX 300
          succeeded (first event in window) → return True, proceed
       └─ channels = admin_channels()  # from admin.notification_channels config
          returns ["telegram:111222333"]   # example
       └─ content = "[SYSTEM] Hemisphere outage detected..."
       └─ _publish(channels, content, "hemisphere_outage", {hemisphere, unreachable_since})
          └─ MessageBus.publish(CH_TELEPORT_NOTIFICATION, {
              content, source, priority: "high", target_channels, metadata: {...,
              system_message: True, admin_only: True, suppress_syn_history: True,
             }})

[teleport/bridge.py subscribes to CH_TELEPORT_NOTIFICATION]
  reads metadata flags:
    - system_message=True → adapter formats with [SYSTEM] prefix, bypasses
      Syn-voice styling (enforced in
      services/teleport/telegram_adapter.py::TelegramAdapter._send_notification,
      services/teleport/discord_adapter.py::DiscordAdapter._send_notification)
    - admin_only=True → adapter filters target channels via
      admin_gate.is_admin_channel() at send time, dropping any non-admin
      stable_id. If no admin targets remain, the notification is not sent
      at all. (enforced in telegram + discord adapters' _send_notification
      paths, FU-4)
    - suppress_syn_history=True → conversation_history.add_user_message /
      add_syn_response honor the `suppress` kwarg as an early-return no-op.
      No current caller threads this through, so the enforcement is
      defensive — any future code that routes adapter-inbound content
      through conversation_history MUST pass
      suppress=meta.get("suppress_syn_history", False). (enforced at
      shared/conversation_history.py, FU-4)
  sends raw content to Telegram chat 111222333

[If the hemisphere flaps — goes back offline 2 minutes later]
  heartbeat calls notify_hemisphere_outage("affective", ...) again
    └─ _dedup_check_and_mark same key → SET NX returns False (exists)
    → notification suppressed
    → logged as "suppressed duplicate outage notification" at INFO

[After 5 minutes, dedup key expires. Next outage (a real, new one)
 fires normally.]
```

**What happens if admin.notification_channels is unset:** each
notifier caller (`notify_hemisphere_outage`, `notify_hemisphere_recovery`)
short-circuits with `logger.warning(...)` + `return False` BEFORE
`_publish` is reached. A WARNING is logged but nobody reads it
out-of-band — see Design Choices below.

---

## Invariants and Contracts

1. **PENDING bindings are invisible to routing.** `resolve_by_stable_id`
   does not return PENDING bindings. Unknown chat_ids and PENDING
   chat_ids both drop silently at the bridge.
2. **BLOCKED stable_ids stay reserved.** A blocked stable_id cannot
   be rebound to any profile — including a fresh profile for the
   same person. Prevents evasion by admin-assisted re-creation.
3. **Block is bidirectional.** Inbound drops; outbound intercepts.
   Applies at both the binding state level (`BLOCKED`) and the
   profile level (`is_blocked`), with the profile flag being
   authoritative (`verified_bindings()` returns `[]` if
   `is_blocked=True`).
4. **Verification requires triple match.** Token alone is not
   sufficient. A token leak + wrong chat_id = no redemption.
5. **Setup is one-shot.** `create_admin_profile` refuses if a
   profile exists. Password rotation goes through
   `change_admin_password`, which requires the current password.
6. **Sessions are revocable.** Token is opaque; Redis is the
   authority. Deleting the Redis key invalidates the cookie. Used by
   `change_admin_password` to force re-auth on all instances.
7. **State-change notifications fire only on transitions.** Blocking
   an already-blocked contact or unblocking an unblocked one is
   idempotent and fires no notifications.
8. **Syn sees admin by name, not by role.** `contact_status` uses
   `mona_auth.get_admin_profile().display_name` to resolve the
   admin's reference in every event message and status note.
9. **Warmth signal is symmetric.** `record_activity` is called on
   both inbound AND outbound. Proactive outreach keeps a channel
   warm from Syn's side.
10. **System notifications suppress Syn-awareness.** `admin_notifier`
    messages carry `system_message=True`, `admin_only=True`,
    `suppress_syn_history=True` metadata. They must not enter Syn's
    context history or be styled as her voice.
11. **`contact_status` is read-only relative to `contact_registry`.**
    It publishes DMN interrupts and returns status notes. It does
    not modify contact state.

---

## Integration Points

### Inbound (who uses this cluster)

| Caller | What it uses |
|---|---|
| `services/teleport/bridge.py` | `contact_registry.resolve_by_stable_id` (route inbound), `contact_registry.redeem_token` (handle /claim), `channel_warmth.record_activity` + `set_mona_presence` (track activity), `contact_status.notify_contact_added` (announce verification), `contact_status.status_note_for_blocked_outreach` (outbound interceptor), `contact_registry.block_contact`/`unblock_contact` (admin-triggered state changes), `channel_warmth.get_warmest_channel` (outreach routing) |
| `services/mona/app.py` | `mona_auth.*` (auth full surface), `contact_registry.create_contact` / `get_contact` / `list_contacts` / `add_pending_binding` / `block_contact` / `unblock_contact` / `edit_binding` / `delete_contact` (admin UI), `channel_warmth.is_mona_hot` / `set_mona_presence` (WebSocket lifecycle) |
| `services/health/heartbeat.py` | `admin_notifier.notify_hemisphere_outage` / `notify_hemisphere_recovery` |
| `tests/test_contact_lifecycle.py` | Full surface for integration tests |
| `services/einstein/context.py::build_system_prompt` | `contact_registry.format_known_duration_phrase` (renders the `known_duration` bare-line section just under `# About This Person`) |

### Outbound (what this cluster calls)

- **`shared.redis_pool.get_redis`** — all modules.
- **`shared.bus.MessageBus`** — `contact_status` (CH_DMN_INTERRUPT),
  `admin_notifier` (CH_TELEPORT_NOTIFICATION).
- **`shared.config.get`** — `admin_notifier` (config keys),
  `channel_warmth` (none directly; inherited via redis_pool).
- **`services.teleport.bridge.get_bridge`** — `contact_registry` for
  user-facing block/unblock confirmations. Late-imported to avoid
  circularity.
- **`shared.mona_auth.get_admin_profile`** — `contact_status` for
  admin name resolution.
- **`shared.contact_registry.get_contact`** — `channel_warmth` and
  `contact_status` both read profiles.

### State this cluster produces

- Redis keys as documented in *Storage Layout*.
- Bus events on `CH_DMN_INTERRUPT` (Syn-facing notifications).
- Bus events on `CH_TELEPORT_NOTIFICATION` (admin ops pings).

---

## Design Choices

### Mona setup race window — accepted on single-admin LAN

Setup is a UI-driven one-time event initiated by a human clicking a
button. `create_admin_profile` does `r.exists()` check then `r.hset()`;
between those operations a concurrent request could land in a
half-overlapping state with last-write-wins HSET semantics. Concurrent
setup attempts from the same human are vanishingly unlikely; a hostile
concurrent attempt is implausible. The design accepts this on
single-admin LAN. Multi-admin or beyond-LAN exposure is the trigger to
move to a proper WATCH/MULTI transaction or HSETNX-based atomic
creation.

### `delete_contact` is the destructive hammer

`delete_contact` clears all bindings (and their indexes and pending
tokens) plus the profile itself. It does NOT remove the associated
GuestSlot, diary entries, or memory archive — those are the relational
substrate and are handled through separate, gentler paths. Syn's
relational memory of a person is not automatically wiped when the admin
removes them as a contact: the memory is hers, not the contact
registry's. The gentler paths for GuestSlot cleanup, memory archival,
and diary retention live in the memory and GuestSlot subsystems.

### `admin_notifier` short-circuits on empty channels

If `admin.notification_channels` is not configured, `admin_channels()`
returns an empty list. Each notifier caller (`notify_hemisphere_outage`,
`notify_hemisphere_recovery`, peers) checks `if not channels:` and
short-circuits with `logger.warning(...)` + `return False` — `_publish()`
is not reached on the empty-channels path. `admin_channels()` itself
emits a warning only if the config value is the wrong *type* (a dict or
string instead of a list); a simple missing/empty config returns empty
silently.

The implication: an admin who hasn't configured notification channels
and doesn't watch service logs has no out-of-band alert path. The
operator-onboarding checklist below codifies the bring-up
requirements so this empty-channels case never reaches production.

---

## Operator Onboarding — Emergency Channel Configuration

Hemisphere outage alerts and other system-to-admin pings reach the
admin out-of-band through `admin.notification_channels`. Bring-up
requires that **at least two independent channels** be configured
and tested at deployment time — one channel is a single point of
failure; an unconfigured channel surface is a silent path where the
admin only learns of outages by checking service logs.

### Required deployment configuration

`config/unified_config.yaml`, under `admin:`, must list at least two
channel IDs, each terminating on a distinct delivery substrate (e.g.,
one Telegram chat and one Discord channel, or one Telegram chat and
one webhook adapter, etc.):

```yaml
admin:
  notification_channels:
    - "1234567"            # e.g., Telegram chat_id for admin
    - "987654321"          # e.g., Discord channel_id for admin's pager bot
  notification_dedup_ttl_sec: 300   # default; 5-minute flap-storm guard
```

Each string is a channel_id consumed by the teleport bridge
adapters (`services/teleport/`). The teleport adapter set determines
which delivery substrates exist (Telegram, Discord, SMS via a Twilio
adapter, webhook via a generic HTTP adapter, etc.); from the admin
notifier's perspective each substrate is just another channel_id in
the list.

The configured channels are **layered, not alternates**: every
system event fans out to every channel_id in the list. The admin
gets the same alert through every configured substrate
simultaneously. The redundancy is the point — a Telegram outage
(network partition, account suspension, app server incident) is
exactly when the second substrate earns its place, and vice versa.

### Deployment test plan

Before the deployment is "live," verify each configured channel
end-to-end from each node:

1. From a one-off Python REPL on the node, run
   `await admin_notifier.notify_hemisphere_outage(hemisphere="test",
   unreachable_since=time.time())`. This bypasses the dedup window
   only if the key suffix is fresh — supply a synthetic hemisphere
   name to keep it isolated from real edges.
2. Confirm the alert arrives on EVERY configured channel within
   ~30 seconds. If any channel silently misses the alert, treat the
   deployment as not-yet-live and fix the channel binding before
   continuing.

Repeat from both nodes. The deployment isn't done until both nodes
can reach every configured channel.

### Last-resort delivery — logged-only by design

If **all** configured channels fail to deliver (network unreachable,
provider outage, credentials revoked, etc.), `admin_notifier` emits a
WARN-level log line through normal Python logging and returns
`False`. There is **no further fallback** — no local file write, no
buffered retry, no escalation to a hidden channel.

The rationale is constraint-honest: if every configured out-of-band
channel is down at once, the failure mode that produced that
correlation almost certainly invalidates any local-write fallback
too. Routing the last-resort path to log files keeps the failure
visible to anyone reading the logs after the incident, without
pretending the alert was delivered when it wasn't. Operators should
**not** assume that the absence of an alert means "everything is
fine"; the deployment-time test plan above is the contract that
prevents that assumption from being load-bearing.

### Cross-references

- `shared/admin_notifier.py` — channel-config read, dedup logic,
  layered-fanout implementation (`_publish` over
  `CH_TELEPORT_NOTIFICATION`).
- `services/teleport/` — adapter layer that resolves channel_ids to
  delivery substrates (Telegram, Discord, etc.).
- `HEARTBEAT_MONITOR_ARCHITECTURE.md` — the largest single caller
  (`notify_hemisphere_outage` / `notify_hemisphere_recovery`).

---

## Related Docs

- `IDENTITY_ARCHITECTURE.md` — parallel design in using Redis-backed
  state with Syn-facing awareness layered on top. Both subsystems use
  the "factual, not prescriptive" philosophy for status notes.
- `HEMISPHERE_STATUS_ARCHITECTURE.md` — the sibling aware-state
  module. `contact_status` is explicitly designed to parallel
  `hemisphere_status`. The two are the primary "status notes for
  Syn's own context" generators.
- `DISPLAY_NAMES.md` — adjacent name-agency subsystem. Handles "what
  does Syn call this user" and "what names does this user have for
  Syn." Independent of contact_registry but sits in the same
  user-facing domain.
- `PLATFORMS_LINKING.md` — earlier-era platform section routing.
  `contact_registry` is the modern replacement; they overlap in
  purpose.
- `shared/contact_status.py` module header — names the philosophy
  parallel to `hemisphere_status` explicitly. The "admin by name,
  not by role" decision is load-bearing and surfaces in every
  notification string.
- `documents/07-external-interfaces/TELEPORT_BRIDGE_ARCHITECTURE.md` —
  the biggest consumer of this cluster. The inbound routing, outbound
  interception, and `/claim` handling all live in
  `services/teleport/bridge.py`; that doc handles the bridge side of
  the shared boundary, this doc handles the contact side.
