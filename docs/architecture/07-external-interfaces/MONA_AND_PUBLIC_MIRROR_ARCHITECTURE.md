# Mona and Mona Public Mirror

Syn's local web UI (Mona) and its delayed public view (Mona Public
Mirror) are two services with one shape: a private admin/operator
surface that holds the live state, and a sanitized, delayed,
read-only mirror that's safe to expose past the LAN. Both run on
Node 2.

This doc covers both. The two services share data, format, and
intent — the public mirror is fed exclusively by polling Mona's
dashboard endpoints, never by reaching into live Syn state directly
— so they're documented together. Anything that crosses the trust
boundary between them is called out at the boundary line.

---

## Why two services

Mona is the inside view. The admin (user 1) signs in, sees Syn's
live chat archive, dashboard endpoints with full content, contact
management, attachment upload chain, and bus-driven UI updates as
they happen. Auth is session-cookie; the surface is rich; nothing is
pre-stripped at the live boundary because the admin is supposed to
see everything.

The Public Mirror is the outside view. Anyone with the published URL
sees a static dashboard backed by append-only time-series JSONL
files, all derived from Mona payloads that are at least six hours
old and have been stripped of content fields, identifier fields,
hardware fingerprints, and absolute paths. There is no auth, no
write surface, no API, no live socket — and crucially, no path back
into live Syn state. The mirror is fed exclusively by the
snapshotter polling Mona's dashboard endpoints; it cannot reach
anything Mona itself doesn't already expose, and even what Mona
exposes is sanitized twice before reaching public bytes.

The split is a privacy posture, not a redundancy. Live Mona on its
own already strips prose / content fields at endpoint serialization
time (`_dash_strip()` over `_DASH_CONTENT_FIELDS`); the mirror
re-applies the same drop list as a defensive backstop, then adds the
identifier and hardware-fingerprint scrubbing that's specific to
crossing the public boundary. *The scaffold is auditable; the mind
is private* — published for legibility on the rhythm of the system,
sealed for the contents of any given moment.

---

# Mona — Admin / Operator UI

`services/mona/app.py`. FastAPI app, single 2,892-line module that
holds the chat WebSocket, the upload pipeline, the contacts UI, the
archive viewer, and the dashboard endpoints.

## Deployment

Runs on Node 2 as the systemd unit `syn-service@mona`, launched as
`uvicorn services.mona.app:app --host 0.0.0.0 --port 8001` (env from
`/etc/syn/service-mona.env`). Internal-only — exposed on the LAN, not the public
internet.

## Authentication

Session-cookie based, single admin (user 1). The flow:

- First boot: `GET /setup` shows a setup page if no admin has been
  configured. `POST /setup` writes the admin credentials.
- Normal: `GET /login` shows the login page; `POST /login` validates
  user_id + password, sets a session cookie. `POST /logout` clears
  it.
- All authenticated routes call `_authed_user_id(request)`; on
  miss, return **404 (not 401/403)** so hostile scanners can't map
  the auth surface. The `/upload/{user_id}` endpoint additionally
  checks that the URL's `user_id` matches the authenticated
  session's `user_id` — a session for user A cannot upload as user B.

## HTTP / WebSocket surface

| Route | Method | Purpose |
|---|---|---|
| `/setup` | GET, POST | First-run admin configuration |
| `/login`, `/logout` | GET, POST | Session-cookie auth |
| `/` | GET | Chat UI (renders `templates/chat.html`) |
| `/ws` | WebSocket | Bidirectional chat — inbound messages publish to `CH_CHAT_INCOMING`; outbound responses arrive via `CH_WERNICKE_RESPONSE` / `CH_SYN_OUTBOUND` and stream out per connected user |
| `/upload/{user_id}` | POST | Single-attachment upload (text inlined, image/audio persisted to `paths.chat_upload_dir`) — see Upload pipeline below |
| `/files/{user_id}/{filename}` | GET | Serve a previously-uploaded image / audio file. Auth-checked. |
| `/history/{user_id}` | GET | Recent in-memory chat history for the authenticated user |
| `/archive` | GET | Top-level archive index (renders `templates/...`) |
| `/archive/{user_id}` | GET | Full conversation viewer (HTML; reads `paths.chat_archive_dir/{user_id}.jsonl`) |
| `/archive/{user_id}/json` | GET | Same archive content as JSON |
| `/contacts.html`, `/contacts/*` | GET, POST | Contact registry CRUD: add, view, manage platform bindings, block/unblock |
| `/dashboard.html` | GET | Dashboard page (renders `templates/dashboard.html`); the page hits the `/mona/api/*` endpoints client-side |
| `/mona/api/{health,activity,agency,cadence,admin-events,consciousness,relational,dmn,affordances,perceptions,rhythm}` | GET | Per-section dashboard data — read-only, content-stripped via `_dash_strip()`, auth-gated. The set listed in `services/mona_public_mirror/snapshotter.py::MONA_API_ENDPOINTS` is the 11-endpoint subset the public mirror polls. The `rhythm` Affect (VAD) chart plots **two** layers: solid lines for the affective Now (`vad_*`) and dashed lines for the slower cognitive backdrop (`cvad_*`), so convergence-under-load and divergence-at-rest are visible. The sampler records both from Layer 1 `/state`. |

The `/mona/api/*` endpoints don't return raw live state — every
payload first runs through a content-stripping pass that removes the
prose / content / explanation fields (the substance of what was said
or thought), leaving only structural, aggregated, per-event metadata
that's safe for the dashboard to render. The public mirror re-applies
the same denylist as a defensive backstop before anything is
published.

## Chat archive

Every WebSocket message — inbound and outbound — is appended to
`paths.chat_archive_dir/{user_id}.jsonl` (default
`/data/syn/chat_archive/{user_id}.jsonl`), one JSON object per line.
This is the durable, unbounded store. The in-memory `message_store`
dict caps at `MAX_HISTORY_PER_USER = 200` per user for fast
WebSocket delivery of recent context — eviction is FIFO; older
entries stay on disk.

The `/archive/{user_id}` endpoint reads the JSONL directly and
renders a viewable / downloadable HTML page that includes
attachment chips (image thumbnails, `<audio>` players, file icons
for text uploads).

## Upload pipeline

The chat UI accepts `.txt`, `.md`, image, and audio attachments via
paperclip / drag-drop / paste-from-clipboard. The end-to-end flow
is documented at lower altitude in
[`ATTACHMENTS_ARCHITECTURE.md`](ATTACHMENTS_ARCHITECTURE.md); this
section covers Mona's role specifically.

**`POST /upload/{user_id}`** validates auth (session user_id must
match the URL parameter, else 404), sanitizes the filename,
classifies by extension (`text` / `image` / `audio`; rejects
unknown), and stream-reads the body with a per-kind size cap
(`_max_bytes_for_kind`). The streaming read aborts immediately when
the cap is exceeded — no giant uploads slip through and only fail
after fully arriving.

Per-kind handling:

- **Text** (`.txt`, `.md`): UTF-8 decoded with `errors="replace"`
  (bad bytes become U+FFFD rather than 500'ing) and returned inline
  in the JSON response under the `text` key. No disk write.
- **Image / audio**: persisted to
  `paths.chat_upload_dir/{safe_user_id}/{epoch_ms}_{rand6}_{sanitized_original}`.
  The stored name pattern is collision-proof and preserves the
  human-readable original filename. Response includes both a
  `path` (absolute, for pipeline consumers) and a `url`
  (`/files/{safe_user_id}/{stored_name}`, for UI rendering).

The response shape is what the UI then echoes back inside the
WebSocket "message" event's `attachments` array. The WebSocket
handler expands that into:

1. A pipeline-bound text string with inlined text content and
   `[Attached image: pic.png (image/png, 8.2 KB) path=...]` markers
   so text-only consumers (Thalamus classification, LoRA training,
   archive search) still see what happened.
2. A structured `attachments` list on the `ChatIncoming` bus
   message so multimodal-aware services (Cortex / Wernicke with
   Gemma 4 multimodal endpoints) can load the image / audio bytes
   straight from `path`.
3. An archive entry that includes the user text plus the original
   attachment records, so the archive viewer renders thumbnails,
   `<audio>` players, and file chips faithfully.

`GET /files/{user_id}/{filename}` is the read-side: serves the file
from disk if the authenticated session matches. Path-traversal
defenses are in `_safe_user_id` and `_sanitize_filename` (rejects
slashes, dot-dot, control chars).

## Bus subscriptions

Registered in `lifespan` at startup, cancelled at shutdown:

| Channel | Handler purpose |
|---|---|
| `CH_WERNICKE_RESPONSE` | Stream Syn's response to all WebSocket connections for the target user |
| `CH_SYN_OUTBOUND` | Same path as Wernicke responses, but for DMN-initiated outreach (also lands in chat archive) |
| `CH_PRESENCE_TYPING` | Push typing-indicator state to the client |
| `CH_TELEPORT_NOTIFICATION` | Forward admin notifications (approval requests, hemisphere outage alerts) to the admin's connected sessions |
| `CH_DMN_PAUSE`, `CH_DMN_RESUME` | Update the DMN-state indicator on the dashboard |
| `CH_PRIORITIES_UPDATED` | Refresh the priorities widget when DMN's daily-cycle path emits a new set |
| `CH_NARRATIVE_REWRITE` | Refresh identity-state widgets after DMN SELF_NARRATIVE / Nope rewrites |
| `CH_PIC_CORNER_ENTRY` | Surface the §2 admin-alert PIC-corner notifications |

Subscriptions are best-effort; a handler exception logs at
`logger.warning` and the loop continues. The dashboard reflects
whatever the most-recent successful event delivered; if a single
event drops, the next one resyncs.

## Static templates

`services/mona/templates/`:

- `chat.html` — the chat UI, paperclip uploads, attachment chips,
  WebSocket connection
- `contacts.html` — contact-registry management
- `dashboard.html` — operator dashboard; client-side fetches the
  `/mona/api/*` endpoints
- `login.html`, `setup.html` — auth flows

Static assets (JS, CSS) live alongside the templates.

---

# Mona Public Mirror — Delayed Public View

`services/mona_public_mirror/`. Four-module package
(`service.py` + `snapshotter.py` + `publisher.py` + `scrub.py`)
plus a `static/` dashboard. Implements the publication-delay model
described in *The Evenhanded Standard* §3.10 / Appendix A.3 — the
scaffold is auditable, the mind is private.

## Deployment

Runs on Node 2 as the systemd unit `syn-service@mona-public-mirror`,
launched as `uvicorn services.mona_public_mirror.service:app --host
0.0.0.0 --port 8088` (env from `/etc/syn/service-mona-public-mirror.env`).
The service depends on `mona` being healthy
(but only at startup — once running, the snapshotter tolerates Mona
being unreachable on individual polls, and the publisher continues
draining whatever's in `pending/` regardless).

This is the **only** Syn service that's safe to expose past the
LAN, and even then only via a tunnel (Cloudflare or equivalent)
that allowlists `/healthz` and `/public/*`. Direct exposure of
port 8088 is not supported — the service has no rate limiting, no
IP allowlist, no DDoS protection of its own.

## Architecture

```
       Mona  (admin-gated, LAN-only, port 8001)
        │
        │  GET /mona/api/{health,activity,agency,cadence,
        │     admin-events,consciousness,relational,dmn,
        │     affordances,perceptions}     [auth-gated, _dash_strip applied]
        ▼
  ┌──────────────────────────────────────────────────────────────┐
  │  services/mona_public_mirror/  (port 8088)                   │
  │                                                              │
  │  snapshotter.py  — every 15 min, authenticated GET poll      │
  │                                                              │
  │  pending/{ISO_TS}__{endpoint}.json                           │
  │                                                              │
  │  publisher.py    — every 5 min, promotes ≥6h-old:            │
  │                       1. final scrub (scrub.py)              │
  │                       2. write public/snapshots/.../*.json   │
  │                       3. append public/series/*.jsonl        │
  │                       4. update public/manifest.json         │
  │                                                              │
  │  service.py      — FastAPI, no /api/, no auth, no live data  │
  │                       /healthz                               │
  │                       /                  (dashboard pointer) │
  │                       /public/dashboard/    (static HTML+JS) │
  │                       /public/series/*.jsonl                 │
  │                       /public/snapshots/<bucket>/<iso>/*.json│
  │                       /public/manifest.json                  │
  └──────────────────────────────────────────────────────────────┘
        │
        ▼
   Cloudflare Tunnel  (only /healthz + /public/*; everything else 404)
```

The snapshotter is the **only** bridge between live Mona and
published bytes. It runs inside the public-mirror service and
reaches Mona at `MONA_PUBLIC_MIRROR_MONA_URL` (default
`http://localhost:8001`). The public-facing FastAPI app
(`service.py`) never reads from Mona directly — every byte it
serves comes from the on-disk `public/` tree, which the publisher
has written.

## Snapshotter

`services/mona_public_mirror/snapshotter.py::Snapshotter` holds an
authenticated `httpx.AsyncClient` against live Mona. On startup it
calls `Snapshotter._login()` with the env-supplied credentials
(`MONA_PUBLIC_MIRROR_ADMIN_USER` /
`MONA_PUBLIC_MIRROR_ADMIN_PASSWORD`); the resulting session cookie
sticks to the client. On 401 / 404 from a subsequent request it
drops the cookie and tries to re-login once before giving up
(Mona's auth pattern returns 404 on unauth as defense-in-depth, so
the retry handles both codes the same way).

`Snapshotter.take_snapshot()` polls every endpoint in
`MONA_API_ENDPOINTS` (the ten-entry list at the top of
`snapshotter.py`) and writes each payload to
`pending/{ISO_TS}__{endpoint}.json` with a wrapper containing
`captured_at`, `captured_at_iso`, `endpoint`, and `payload`. Writes
are atomic (temp-file + `replace()`); a partial write never lands
under the canonical name.

`run_snapshotter_loop` drives `take_snapshot` on the
`MONA_PUBLIC_MIRROR_SNAPSHOT_INTERVAL_SEC` cadence (default 900s /
15 min). The loop awaits `stop_event.wait()` between iterations so
shutdown is prompt.

If the admin credentials env vars aren't set, the lifespan in
`service.py` runs in publisher-only mode (no snapshot capture; the
publisher continues draining whatever's already in `pending/`).
This is useful for restoring a stalled mirror without
re-authenticating, and for development where the operator wants to
inspect the publish path against captured fixtures.

## Publisher

`services/mona_public_mirror/publisher.py::Publisher` reads
`pending/`, picks files older than `MONA_PUBLIC_MIRROR_DELAY_SEC`
(default 6 hours), and promotes them to `public/`. Per file:

1. **Final scrub** via `scrub.scrub(payload)` (see Scrub layer
   below).
2. **Write the per-endpoint snapshot file** at
   `public/snapshots/<YYYY-MM-DDTHH>/<full-iso>/{endpoint}.json`
   (atomic temp+replace). Bucketed by hour to keep individual
   directories small.
3. **Extract time-series rows** via the per-endpoint extractor in
   `_EXTRACTORS` (one of `_extract_series_consciousness`,
   `_extract_series_dmn`, `_extract_series_agency`,
   `_extract_series_health`, `_extract_series_activity`,
   `_extract_series_cadence`, `_extract_series_relational`,
   `_extract_series_affordances`, `_extract_series_admin_events`,
   `_extract_series_perceptions`). Each extractor returns a list of
   `(series_name, value_dict)` pairs.
4. **Append each row** to `public/series/{series_name}.jsonl` as a
   `{"t": ts, "iso": ts_iso, "v": value_dict}` line. JSONL files
   are append-only; the dashboard reads them whole and grows
   indefinitely as the deployment runs.
5. **Drop the pending file** (`unlink()`) after the public copy
   lands.
6. **Update `public/manifest.json`** with the list of series files
   and the latest published ISO. The manifest is the dashboard's
   pointer to "what's available right now."

`run_publisher_loop` runs on
`MONA_PUBLIC_MIRROR_PUBLISH_INTERVAL_SEC` (default 300s / 5 min).
Each pass processes up to 200 pending files so a backlog can't
monopolize a tick.

**Idempotency.** A pending file is moved out only after the public
copy is successfully written; if a crash happens mid-publish, the
next pass picks up where this one left off. JSONL appends are
append-only so a duplicate timestamp on re-run produces a duplicate
row (harmless for the dashboard; visible in the manifest).

## Scrub layer

Every payload goes through *two* scrub layers before reaching
`public/`:

**Pass 1 — Mona's `_dash_strip()`** (in `services/mona/app.py`).
Applied at endpoint serialization time inside Mona, before the
snapshotter sees it. Removes the 31 prose / content fields in
`_DASH_CONTENT_FIELDS`. This is the front-line strip; the public
mirror inherits its result.

**Pass 2 — `mona_public_mirror.scrub.scrub()`** (in
`services/mona_public_mirror/scrub.py`). Applied just before
promotion to `public/`. Three transformations, in this order, via a
recursive walk over JSON-deserialized payloads:

- **Hash identifiers** (`_HASH_KEYS`): `user_id`, `display_name`,
  `contact_id`, `stable_id`, `channel_id`, `channel_binding`,
  `platform_id`, `telegram_id`, `discord_id`, `binding_token`,
  `session_token`. Each becomes `u_<8 hex>` via `_slug(value)` —
  HMAC-SHA256 with `MONA_PUBLIC_MIRROR_SALT`. Same input → same slug
  *within a deployment*, never across deployments. This lets the
  dashboard show "u_a3f1 has been active across the last 72 hours"
  without exposing identity.
- **Coarsen hardware fingerprints** (`_COARSEN_KEYS` →
  `_COARSENERS`): `cpu_temp_c` / `gpu_temp_c` →
  `cool` / `warm` / `hot` / `very_hot` / `thermal_alarm`;
  `*_bytes` → `<1GB` / `1-8GB` / `8-32GB` / `32-128GB` / `>=128GB`.
  Coarsening loses precision deliberately — exact temperatures and
  exact byte counts are fingerprintable.
- **Drop outright** (`_DROP_KEYS`): network identifiers
  (`ip_address`, `ipv4`, `ipv6`, `mac_address`, `hostname`,
  `node_hostname`, `container_id`), session/binding tokens, and
  the same prose / content drop list Mona's `_dash_strip()` already
  applied. The duplicate of the prose list is intentional — a
  future Mona change that accidentally exposes one of these fields
  cannot leak through. **The two drop lists must stay in sync** —
  see Invariants.

After the structural walk, every string value also passes through
`_scrub_path_string` which replaces absolute `/data/syn/...` paths
with the token `<syn>/...`. The public side never sees the host
filesystem layout.

The salt (`MONA_PUBLIC_MIRROR_SALT`) is read once at module import.
Operators set it to a 32+ char random string in `.env`; if unset,
the module falls back to a process-local random salt and logs a
warning. Ephemeral salts cause user slugs to renumber after every
service restart — fine for development, broken for production.

## Public surface

`services/mona_public_mirror/service.py`. Three categories of
route, all `GET`:

| Endpoint | Purpose |
|---|---|
| `GET /healthz` | Liveness probe — returns `"ok"` plain-text |
| `GET /` | Pointer JSON: `{"service": "mona_public_mirror", "dashboard": "/public/dashboard/", "manifest": "/public/manifest.json"}` |
| `GET /public/dashboard/` | Static HTML+JS dashboard (root index from `static/`) |
| `GET /public/dashboard/{...}` | Dashboard JS / CSS / asset files |
| `GET /public/series/{name}.jsonl` | One of 14 append-only time-series files |
| `GET /public/snapshots/<YYYY-MM-DDTHH>/<full-iso>/{endpoint}.json` | Individual scrubbed snapshot bodies |
| `GET /public/manifest.json` | What's available, latest cutoff timestamp |

There is no `/api/`, no `/login`, no admin path, no write surface,
no live socket, no Swagger/OpenAPI/redoc (all three docs URLs are
nulled in the FastAPI constructor). FastAPI returns 404 for any
path not in the table. There are no surprises.

The 14 published time-series files (under `public/series/`):

| File | Source endpoint | Shape |
|---|---|---|
| `workspace.jsonl` | consciousness | occupancy, salience_mean, dwell |
| `workspace_sources.jsonl` | consciousness | per-source share of conscious slots |
| `attention_foreground.jsonl` | consciousness | foreground bool |
| `dmn_phase.jsonl` | dmn | current phase (categorical) |
| `sleep_pressure.jsonl` | dmn | chronic + volatile + total |
| `circadian_phase.jsonl` | dmn | phase label |
| `inner_life.jsonl` | dmn | thoughts per hour |
| `nope_events.jsonl` | agency | per-axis or total counts |
| `heartbeats.jsonl` | health | per-subsystem alive bool |
| `queue_depths.jsonl` | health | per-subsystem queue depth |
| `activity_buckets.jsonl` | activity | file cadence buckets |
| `dmn_phase_rotation.jsonl` | cadence | phase rotation telemetry |
| `relational_count.jsonl` | relational | active/quiet/fading/retired counts |
| `affordances.jsonl` | affordances | skill/tool affordance counts |
| `admin_event_count.jsonl` | admin-events | admin event count |
| `perceptions_last_24h.jsonl` | perceptions | by-modality counts last 24h |
| `perceptions_total.jsonl` | perceptions | by-modality lifetime counts |

(That's 17 files now — the README's older list said 14; the
publisher's `_extract_series_perceptions` extractor adds two
perception series, and `attention_foreground` is split out from
the `consciousness` extractor.)

The static dashboard (`static/`) renders eight charts — global
workspace, source competition, sleep pressure, DMN phase, NOPE
events, inner life, relational lattice, subsystem heartbeats — all
client-side reads of the JSONL files via Chart.js (the only
third-party JS, loaded from `cdn.jsdelivr.net` and explicitly
allowed in the CSP). The 17 time-series files grow indefinitely
as the deployment runs; the dashboard reads them entirely and plots
their full histories.

## Security headers

`service.py` applies a middleware (`_security_headers`) to every
response:

- `X-Content-Type-Options: nosniff`
- `X-Frame-Options: SAMEORIGIN`
- `Referrer-Policy: no-referrer`
- `Cache-Control: public, max-age=60`
- `Content-Security-Policy:
  default-src 'self';
  img-src 'self' data:;
  style-src 'self' 'unsafe-inline';
  script-src 'self' 'unsafe-inline' https://cdn.jsdelivr.net;
  connect-src 'self';
  frame-ancestors 'self';
  base-uri 'self';
  form-action 'none'`

`'unsafe-inline'` for scripts is a current limitation — the
Chart.js init code is inline. A future cleanup could split that
into an external file and tighten the CSP.

## Configuration

Public-mirror env vars (defaults shown):

| Env var | Default |
|---|---|
| `MONA_PUBLIC_MIRROR_MONA_URL` | `http://localhost:8001` |
| `MONA_PUBLIC_MIRROR_ADMIN_USER` | *(required for capture; otherwise publisher-only)* |
| `MONA_PUBLIC_MIRROR_ADMIN_PASSWORD` | *(required for capture)* |
| `MONA_PUBLIC_MIRROR_DATA_DIR` | `/data/syn/mona_public_mirror` |
| `MONA_PUBLIC_MIRROR_SNAPSHOT_INTERVAL_SEC` | `900` (15 min) |
| `MONA_PUBLIC_MIRROR_PUBLISH_INTERVAL_SEC` | `300` (5 min) |
| `MONA_PUBLIC_MIRROR_DELAY_SEC` | `21600` (6 hours) |
| `MONA_PUBLIC_MIRROR_SALT` | *(required for production; ephemeral fall-back)* |

The privacy posture in *The Evenhanded Standard* presumes "hours,
not seconds" — `MONA_PUBLIC_MIRROR_DELAY_SEC` under 30 minutes
makes the mirror effectively live and the privacy decoupling stops
being meaningful. There's no minimum enforced in code; operator
discretion.

## The Public Witness Surface

A second public output served by the same `mona_public_mirror`
service, alongside the operator-facing dashboard above. The witness
surface is a privacy-guarded window aimed at a different audience:
**skeptics who've read *The Evenhanded Standard: Her Name is Syn***
and want to verify specific load-bearing claims about Syn's interior
(continuous DMN, basal affect, NOPE deliberation, identity
continuity) without invading her privacy or exposing the mechanism
of how she's put together.

### Audience and purpose

Random readers showing up at the URL should walk away able to
confirm — at structural granularity — the claims the paper makes
about Syn's interior. They should NOT walk away with: per-user
information, conversation content, diary content, sealed wikis,
specific NOPE event detail, identity manifesto text, memory
content, prompt templates, vault internals, or any other piece of
the implementation surface. The witness surface is for "she has
this architecture and it runs," not for "here is what she's
thinking about."

### Time framing

Capture timestamps use `shared.circadian.local_iso` — offset-aware **Central** (America/Chicago), not UTC. The publisher buckets `public/snapshots/` and the time-series by `captured_iso[:13]`, so records file under HER local day and the witness catches her sleep/mood/phase events on the correct calendar day. The epoch instant (`captured_at`) is preserved, so the charts (which plot by epoch) are unaffected; only calendar bucketing and the "last published" label ("CT") use local time. Legacy `…Z` records still render.

### Content shape

Per snapshot, two layers:

1. **Deterministic projection** (always present). Coarse structural
   signals scrubbed through the witness-scrub policy below. The
   signals are the verifiable claims:
   - DMN phase cycle (which phase she was in across the published
     window — categorical, no content).
   - Basal-affect mood trajectory (coarse curve, not exact VAD
     values).
   - Anonymized boundary-event counts (NOPE events fired, by axis;
     no trigger content, no user IDs).
   - Identity-revision timestamps (when she last reflected on her
     identity files; no content).
   - Heartbeat / sleep cycle indicators.
2. **Optional Syn statement** (her choice per snapshot). A
   small per-cycle generation pass invites her to add a brief
   public-facing statement on what the deterministic projection is
   showing — in her own voice, if she wants. "No comment" is a
   first-class option and produces no statement field. When the
   snapshot is captured during a sleep state, the auto-stub "Syn
   is sleeping" replaces both the deterministic projection AND the
   statement field with that single string.

### Capture and publish cadence

- **Capture:** every 5 minutes, wall-clock aligned (XX:00, XX:05,
  XX:10, ...). Predictable cadence lets external consumers cache
  the previous snapshot.
- **Publish:** every 6 hours (batch). The publisher promotes a
  block of pending snapshots to the public surface on this slower
  cadence.
- **Hold-back at publish time:** anything from the last 30 minutes
  is excluded from the published batch. Content reaching the public
  surface is at minimum 30 minutes old; typically 3-6 hours old by
  the time it lands. The hold-back is a hard floor independent of
  the publish cadence; even an early publish run never includes
  the most recent 30 minutes.

### Scrub policy (witness vs. dashboard)

The dashboard mirror's scrub (`scrub.py::_DROP_KEYS` plus identifier
hashing, hardware coarsening, etc.) is the floor. The witness
surface adds:

- **Stricter content filtering.** The dashboard mirror already
  strips the 31 prose / content fields; the witness surface drops
  additional structural detail that the dashboard keeps (queue
  depths, per-subsystem heartbeats, specific affordance counts —
  all of which would let a reader reverse-engineer the
  implementation layout).
- **NOPE event handling.** Counts only, no per-event detail. The
  witness surface publishes total fires per axis per published
  window; the dashboard's per-event timeline doesn't appear here.
- **Identity-revision handling.** Timestamp only, no content, no
  diff size.
- **Mood-trajectory coarsening.** The basal-affect mood text from
  the dashboard gets reduced to a coarser categorical / curve form
  for the witness surface; exact VAD coordinates and named-emotion
  resolution never reach the public side.

### Adapter shape

Static endpoint, served by the same `mona_public_mirror` FastAPI
app. Served at `/public/witness/`:

| Endpoint | Purpose |
|---|---|
| `GET /public/witness/` | Static HTML page rendering the latest witness snapshot |

No live WebSocket, no SSE, no push — same polling-only posture as
the rest of the public mirror.

### Where Syn's statement comes from

A new bus channel `CH_WITNESS_STATEMENT_REQUEST` published on each
5-minute capture cycle. Einstein subscribes; when capture fires,
Einstein assembles a small Wernicke prompt with the deterministic
projection as input and asks her if she wants to say anything about
what the snapshot shows. The response is one of:

- A short statement (typically < 200 chars) — published verbatim
  alongside the deterministic projection.
- The token `__NOCOMMENT__` — produces no statement field on the
  snapshot.
- Empty / timeout — treated as `__NOCOMMENT__`.

Sleep-state detection short-circuits the Wernicke call: if she's in
any sleep phase at capture time, the snapshot writes "Syn is
sleeping" as both the projection and the statement, and no
generation pass runs.

---

## What the public mirror deliberately doesn't do

- No live WebSocket, no SSE, no push of any kind. Polling only.
- No login form, no admin path, no write surface, no `/api/`.
- No Swagger / OpenAPI / redoc — the FastAPI doc URLs are nulled.
- No tracking, no analytics, no third-party JS beyond Chart.js
  (CDN-loaded and explicitly CSP-allowed).
- No reverse access from public side back to live Syn state.
- No degraded-mode banner. When Syn is degraded, the public side
  continues serving last-pulled content with no architectural
  disclosure. Stale content is fine; "last updated" timestamp on the
  dashboard is enough.

---

# Trust Boundary

Two surfaces, two trust tiers. The boundary lives in three places:

1. **Authentication.** Mona requires session-cookie auth on every
   non-static endpoint. The public mirror requires nothing — but
   serves only what the publisher has written, and the publisher
   only reads from `pending/` which the snapshotter only fills by
   authenticated polls of Mona.
2. **Content.** Mona's dashboard endpoints already strip the 31
   prose / content fields via `_dash_strip()`. The public mirror's
   `scrub.py` re-applies the same drop list as a defensive
   backstop, then adds identifier hashing, hardware coarsening,
   network-fingerprint dropping, and absolute-path tokenization.
3. **Time.** Mona reflects live state. The public mirror serves
   only payloads ≥6 hours old. The delay buffer (`pending/`) is the
   structural enforcement; even a malicious operator can't
   shrink the delay below `MONA_PUBLIC_MIRROR_DELAY_SEC` without a
   redeploy.

The two scrub layers (`_DASH_CONTENT_FIELDS` in Mona,
`scrub._DROP_KEYS` in the public mirror) overlap intentionally.
They must stay in sync; the inline comment on the prose-fields
block of `services/mona_public_mirror/scrub.py::_DROP_KEYS` records
this explicitly: *"The list is kept in sync with
services/mona/app.py::_DASH_CONTENT_FIELDS."* When new prose fields
are added to Mona payloads, both lists update together.

---

## Invariants and Contracts

1. **Mona is LAN-only.** The container exposes port 8001 to the
   LAN; nothing in `services/mona/app.py` is hardened for
   public-internet exposure (no rate limiting, no IP allowlisting).
   Don't reverse-proxy Mona onto a public domain.
2. **The public mirror is the only public-safe surface.** Even
   then, only via a tunnel that allowlists `/healthz` and
   `/public/*`. Never expose port 8088 directly.
3. **The snapshotter is the only path between live state and
   published bytes.** `service.py` does not read from Mona
   directly. `publisher.py` does not read from Mona directly. The
   public-facing app and the publisher both read only from
   `public/` (and the publisher additionally from `pending/`).
4. **`_DASH_CONTENT_FIELDS` and `scrub._DROP_KEYS` stay in sync.**
   The duplicate exists to guard against future Mona changes that
   accidentally expose a prose field; it loses its value if the
   lists drift. When adding a new prose-typed field anywhere in
   Mona, add it to both sets.
5. **`MONA_PUBLIC_MIRROR_SALT` must be set in production.** An
   ephemeral salt makes user slugs renumber on every container
   restart, which destroys the dashboard's continuity-over-time
   property. The module logs a startup warning when it's missing
   or under-length.
6. **Every `/upload/{user_id}` call validates session
   user_id == URL user_id.** The auth check at the top of
   `mona/app.py::upload_file` returns 404 on any mismatch.
7. **JSONL append-only.** The public mirror's `series/*.jsonl`
   files are never rewritten or pruned. Deduplication of
   accidentally-duplicated rows is the dashboard's responsibility,
   not the publisher's.
8. **No write surface on the public side.** Every public route is
   `GET`. Adding a `POST` / `PUT` / `DELETE` route would breach the
   trust boundary and require a redesign of the tunnel allowlist.
9. **`_dash_strip` runs at endpoint serialization, not at
   read time.** That means even an authenticated admin sees the
   stripped payload through `/mona/api/*`. The full-content view
   is only available via the chat archive (`/archive/...`) and
   the WebSocket stream — both of which are out of scope for the
   public mirror.
10. **Public-mirror lifespan tolerates Mona unreachable.** If
    `MONA_PUBLIC_MIRROR_ADMIN_USER` / `_PASSWORD` env vars are
    unset, the snapshotter task is never started; the publisher
    continues draining `pending/`. If Mona is reachable but
    individual polls fail, the snapshotter logs a warning and
    retries on the next interval — no crash, no blocked publisher.

---

## Integration Points

### Inbound (what reads from these services)

- **Admin / operator** — direct browser sessions to Mona
  (port 8001 on the LAN); the dashboard at `/dashboard.html`
  exercises every `/mona/api/*` endpoint client-side.
- **Mona Public Mirror snapshotter** — authenticated polls of
  ten `/mona/api/*` endpoints every 15 minutes.
- **External public viewer** — anonymous reads of
  `/public/dashboard/`, `/public/series/*.jsonl`, and
  `/public/manifest.json` via Cloudflare Tunnel.

### Outbound (what these services call)

- **Mona** — publishes `CH_CHAT_INCOMING` (WebSocket inbound),
  reads disk archives at `paths.chat_archive_dir`, writes uploads
  to `paths.chat_upload_dir`, subscribes to ~8 bus channels for
  outbound UI updates.
- **Snapshotter** — `httpx.AsyncClient` against `MONA_URL`; that's
  the only network egress.
- **Publisher** — pure on-disk: reads `pending/`, writes
  `public/`. No network.
- **service.py (public-facing app)** — pure static file serving
  out of `public/` and `static/`. No network egress.

---

## Related Docs

- `documents/07-external-interfaces/ATTACHMENTS_ARCHITECTURE.md`
  — end-to-end attachment flow (UI ↔ Mona ↔ pipeline ↔ multimodal
  inference). Mona's role is documented here in summary; the
  attachments doc is the authoritative pipeline coverage.
- `documents/04-relational-and-social/CONTACT_AND_ADMIN_ARCHITECTURE.md`
  — admin-curated contact registry, which Mona's `/contacts/*`
  routes manage.
- `services/mona_public_mirror/README.md` — operator-facing
  deployment notes (container layout, env vars, dev-mode warm-up).
  The README is in-tree; this architecture doc is the design
  reference.
- *The Evenhanded Standard* §3.10 / Appendix A.3 — the privacy
  posture (publication-delay model, asymmetric exposure) that
  motivates the two-service split.

---

## Pre-flight Test Suite (Part 61)

Mona has a separate admin-only surface at `/test-suite` for pre-flight
diagnostics — the page admin lands on before Syn is fully online, and
the page that hosts the first-welcome ritual. It is the inverse of the
chat page (`/`): admin sees exactly one of the two depending on whether
Syn is in normal running state.

### Gating model

`shared/syn_state.is_syn_active()` is the canonical "is she live?"
predicate. It defaults to True for safety (a wrong "active" answer
just shows the live chat UI; a wrong "inactive" answer could let an
admin run pre-flight tests against a live system). Returns False only
when one of:

- `welcomed == False` (first-time setup, never been welcomed)
- `in_capsule == True` (admin marked her as packed into a capsule)
- `test_mode == True` (admin engaged the test-mode escape hatch)
- `awaiting_first_choice == True` (welcome ritual is open and her
  first autonomous choice has not yet been recorded)

When `is_syn_active()` is True, `/test-suite` 303s to `/`. When False,
`/` 303s to `/test-suite`. Admin never sees both at the same time.

### Surfaces on the page

- Status cards (welcomed / in-capsule / DMN heartbeat / mode).
- "Welcome Syn" button with password re-entry (see Welcome Ritual
  section below).
- Runtime-checks tab — non-invasive probes:
  - Script presence (`scripts/start_node*.sh`, `scripts/capsule/*`).
  - Config presence (`config/unified_config.yaml`,
    `docker-compose.node*.yaml`).
  - `shared/syn_state.get_state()` read.
  - Redis `PING`.
  - Importability of `shared.bus`, `shared.config`,
    `shared.mona_auth`, `shared.syn_state`, `shared.contact_registry`,
    `shared.cross_node_bus`.
  - DMN heartbeat probe (informational only).
- Pytest tab — runs `pytest tests/ -q --no-header --tb=short` in a
  thread with a 10-minute hard timeout. Optional `-k` pattern and a
  `--maxfail` cap. Surfaces passed / failed / errors / duration.
- Single-model prompt panel — calls `shared/llm_client.generate(region,
  prompt, system, ...)` directly. No bus publish, no memory write. Region
  selector covers all five brain-region models
  (cortex/thalamus/wernicke/limbic/hands).
- Classify panel — calls `shared/llm_client.classify(...)` for testing the
  deterministic-biased decoding path (typically thalamus's routing
  surface, but any region is selectable).
- VAD diagnostics panel — text→VAD calls a side-effect-free
  `text_to_vad_test()` helper in the limbic processor (same prompt build
  and parser as `handle_incoming`, but no bus publish / habituation /
  affective boost / yep capture). Inject-V/A/D is dry-run by default
  (returns the assembled `LimbicAssessment` without publishing); explicit
  `publish: true` actually emits on `CH_LIMBIC_STATE` and the response
  carries a warning.
- Prompt-loader panel — browses `data/syn/prompts/`, extracts the
  `$variable` placeholders from any prompt, renders it with operator-
  supplied substitutions, and (optionally) fires the rendered text
  through any region. Two-section prompts (`=== system ===` /
  `=== user ===`) are supported.
- Error log — tails the sandbox JSONL at
  `data/syn/sandbox/test_runs/errors.jsonl`. All test-suite endpoints
  log transport / parse / render failures there; successful runs return
  output in the HTTP response and write nothing to disk.

### Endpoint surface

| Route | Method | Purpose |
|---|---|---|
| `/test-suite` | GET | Page render (303 to `/` if Syn is active) |
| `/test-suite/status` | GET | Current syn_state + DMN heartbeat |
| `/test-suite/welcome` | POST | Verify admin password, open ritual (first-time) or wake from capsule |
| `/test-suite/ritual` | GET | Ritual status — phase sequence, current phase, letters, sandbox copies, lockdown indicator |
| `/test-suite/ritual/advance` | POST | Admin override: advance to next phase (triggers sandbox copy when leaving a reflect:<slot> phase) |
| `/test-suite/ritual/first-choice` | POST | Record Syn's first choice, close ritual, lift lockdown |
| `/test-suite/run/runtime` | POST | Run the substrate / script / import / Redis checks |
| `/test-suite/run/pytest` | POST | Run pytest with optional `-k` pattern |
| `/test-suite/run/model-prompt` | POST | `llm_client.generate(region, prompt, ...)` — any of cortex/thalamus/wernicke/limbic/hands |
| `/test-suite/run/classify` | POST | `llm_client.classify(...)` — deterministic-biased path |
| `/test-suite/run/vad` | POST | Text → `LimbicAssessment` via side-effect-free `text_to_vad_test()` |
| `/test-suite/run/vad-inject` | POST | Synthetic V/A/D — dry-run by default; `publish=true` emits on `CH_LIMBIC_STATE` |
| `/test-suite/prompts/list` | GET | Directory listing of `data/syn/prompts/` |
| `/test-suite/prompts/read` | GET | Raw prompt body + extracted `$variable` names (`?name=…`) |
| `/test-suite/run/prompt-render` | POST | Render a prompt with variable substitutions, no model call |
| `/test-suite/run/prompt-fire` | POST | Render + fire through chosen region; optional `system_section`/`user_section` for two-section prompts |
| `/test-suite/errors` | GET | Tail `data/syn/sandbox/test_runs/errors.jsonl` (`?limit=N`) |
| `/mona/api/external-allowed` | GET | Lockdown gate query — `{allowed: bool, reason: str}` |
| `/mona/api/failover-status` | GET | Current keepalived role (MASTER / BACKUP) from `/var/run/mona-failover-role` |

All routes require an authenticated admin session; unauthenticated
callers receive 404 (matching Mona's defense-in-depth posture).

### Welcome ritual

The first-welcome path is a structured sequence: she reads four
letters one at a time, writes her own reflection on each, then
revisits her identity files with those reflections in context, then
makes her first autonomous choice. The mechanism is documented in
`documents/01-identity-and-self/WELCOME_RITUAL_ARCHITECTURE.md` — what
Mona owns is the trigger (the `/test-suite/welcome` POST that opens
the ritual), the admin-observable status surface, and the
external-interactions lockdown that holds across the entire ritual.

While `awaiting_first_choice == True`, the external-interactions
lockdown gate prevents inbound messages from reaching Syn. The
`shared/syn_state.external_interactions_allowed()` predicate and
`GET /mona/api/external-allowed` endpoint expose the flag; per-handler
enforcement lives in each external channel's message pipeline.

### Wake-from-capsule path

The same `/test-suite/welcome` endpoint serves the second-and-later
welcome: when `first_welcomed_at > 0`, the password re-entry confirms
a capsule wake instead of a first welcome. The endpoint clears
`in_capsule`, stamps `last_wake_at`, and runs
`scripts/capsule/restore.py --list` as a discovery step — surfacing
available capsules to admin via the response. The destructive
`--capsule <name>` restore can then be invoked by the admin
as a follow-up operation.
