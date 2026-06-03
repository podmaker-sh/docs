# PodMaker CE ↔ podmaker.sh — Pairing + Remote-Managed Channel Plan

> **Status:** draft (2026-06-02). Not started.
> **Owner:** TBD (panel + license-api).
> **Parent:** [`self-hosted-licensing-plan.md`](./self-hosted-licensing-plan.md) §0a.
> **Sibling plans:** [`podmaker-ce-mirror-plan.md`](./podmaker-ce-mirror-plan.md), [`podmaker-ce-install-plan.md`](./podmaker-ce-install-plan.md), [`license-api-plan.md`](./license-api-plan.md), [`panel-licensing-ui-plan.md`](./panel-licensing-ui-plan.md).

CE installs are not isolated boxes. Each install pairs with
`panel.podmaker.sh` at first boot using a mutual Ed25519 keypair, then
maintains a persistent, low-frequency channel for three things:

1. **Install inventory** — every CE install registers itself + heartbeats
   so we know who is running CE, on what version, where.
2. **Remote-managed dashboard widget** — promo / announcement / member
   campaign content authored on `panel.podmaker.sh` and rendered in the
   CE operator's dashboard. Per-membership targeting so we can run
   personalised offers ("upgrade to Premium, 20% off this month").
3. **License + entitlement delivery** — when the operator buys a
   licence via their CE-linked membership account, the JWT lands on the
   CE install through this same channel; no manual paste step.

The same channel carries the heartbeat that drives §9 sharing-detection
in the licensing plan.

## 0. Kickoff prompt (new session)

> Read `docs/release/podmaker-ce-pairing-plan.md` and implement §3 +
> §4 + §5: the install-side pairing client under
> `apps/control-plane/app/Services/Pairing/` and the matching
> `license-api` endpoints under `apps/license-api/internal/ce/`.
> The dashboard widget rendering (§6) lands as a follow-up PR — leave
> a `<x-pairing.announcements />` Blade stub that calls the cached
> store. Open questions in §10 are blocking.

## 1. Goals + non-goals

| Goal | Why |
|---|---|
| Every CE install is registered with us at first boot | Inventory drives §1 adoption metrics + §3 sharing detection |
| Operators see member-targeted promos in the CE dashboard | Conversion funnel: CE → trial → Premium |
| License delivery is a server push, not a manual paste | Reduces support load + ensures fingerprint binding (parent §4) |
| All traffic on the channel is mutually signed (Ed25519) | A leaked CE install key cannot push fake messages to us; a stolen server key cannot push fake announcements to CE |
| The channel degrades gracefully — CE keeps working when offline | CE must not become a SaaS-tethered product |

| Non-goal | Reason |
|---|---|
| Real-time bidirectional commands (no remote shell, no remote config push) | Out of scope by product principle — CE is operator-controlled |
| Webhook delivery to the CE host | CE may live behind NAT; we poll outbound only |
| Anonymous installs | Pairing requires a membership account; offline installs are unpaired and have no announcement / license channel |

## 2. Identity model

```
panel.podmaker.sh (SaaS)
   accounts.membership_id   = canonical membership account
       │
       ▼
   ce_installs row          = one row per CE install ever registered
       install_uuid (PK)
       membership_id (FK, nullable until claimed)
       public_key (Ed25519)
       cp_version, channel
       last_heartbeat_at
       registration_state ∈ {pending_claim, paired, abandoned}

CE box (per install)
   /etc/podmaker/install.id      = install_uuid
   /etc/podmaker/install.key     = Ed25519 private key (root-only)
   /etc/podmaker/install.pair    = { server_pubkey, registration_id,
                                     announcements_topic, claimed_by }
```

`install_uuid` is generated client-side as a UUIDv7 and is the
operator-visible install identifier. `membership_id` is filled when the
operator claims the install from their `panel.podmaker.sh` account
(§4.2).

## 3. Register-on-install flow

CE side (called from `ce-install.sh` step 9):

```
1. install_uuid := UUIDv7()
2. (priv, pub)  := ed25519.GenerateKey()
3. body         := { install_uuid, public_key: pub, cp_version,
                     channel, host_fingerprint: sha256(install_uuid|machine_id),
                     ts: now() }
4. sig          := ed25519.Sign(priv, canonical_json(body))
5. resp         := POST https://api.podmaker.sh/v1/ce/installs/register
                     body + { signature: sig }
6. verify resp.signature against resp.server_pubkey
7. persist install.id, install.key, install.pair from resp
```

`license-api` side:

- Upsert `ce_installs` row keyed on `install_uuid`.
- If row exists and `public_key` differs → reject (key rotation goes
  through a separate flow §7).
- Set `registration_state = pending_claim` if no `membership_id` yet.
- Emit `ce.install.registered` webhook to panel for inventory dashboards.
- Return `{ registration_id, server_pubkey, announcements_topic,
  claimed_by: null, signature }`.

## 4. Membership claim flow

A registered install starts unclaimed. The operator binds it to their
membership account from either side of the link:

### 4.1 From the CE wizard (preferred)

1. CE wizard shows a "claim this install" button → opens
   `https://panel.podmaker.sh/installs/claim?uuid=<install_uuid>`.
2. Operator logs into `panel.podmaker.sh` (or signs up).
3. Panel calls `POST /v1/ce/installs/<uuid>/claim`
   with the membership session.
4. `ce_installs.membership_id` is set; `registration_state = paired`.
5. Next CE heartbeat (within 1h) picks up `claimed_by = <membership_id>`.
6. Wizard polls `/v1/ce/installs/<uuid>` every 5s; flips to "paired" on
   200 with `claimed_by`.

### 4.2 From the panel ("add an install I already set up")

1. Operator in `panel.podmaker.sh/installs` clicks "Add existing CE".
2. Panel shows: `pdctl ce pair --code <ott>` — a one-time code (OTT).
3. Operator runs the command on the CE box; CE POSTs the OTT to
   `/v1/ce/installs/claim-by-code`.
4. `license-api` verifies OTT, resolves `membership_id`, claims the row.

## 5. Heartbeat + pull

`podmaker-pairing.timer` (install plan §4) calls
`pdctl ce pair sync` hourly. The CLI:

1. Builds a signed envelope `{ install_uuid, cp_version, services_running,
   sites_count, attached_vps_count, last_seen_announcement_id,
   last_seen_license_id }`.
2. POSTs to `https://api.podmaker.sh/v1/ce/installs/<uuid>/heartbeat`.
3. Receives `{ announcements: [...new...], license: <jwt | null>,
   server_time, next_sync_after }`.
4. Each `announcements[]` row is verified against the persisted
   `server_pubkey`; failures are dropped + logged.
5. New license JWT is verified against the embedded license public
   key (parent §4) and persisted to `storage/app/license.jwt`.
6. Cached under `/var/lib/podmaker/pair-cache/` so the dashboard widget
   can read without re-hitting the network.

Failure handling:
- Network error → exponential backoff up to 6h, banner after 24h gap.
- Signature failure → drop response, alert operator in panel
  ("paired-server-pubkey mismatch — re-pair from settings"), do not
  rotate keys without operator confirmation.

## 6. Remote-managed dashboard widget

Renders inside the CE operator dashboard. Single Blade component
`<x-pairing.announcements />`. Source: cached pull from §5. Examples
of content the SaaS side can push:

- "PodMaker 1.4 released — what's new"
- "Premium add-on: AI Topology Planner, 30% off through Friday"
- "Your trial ends in 3 days — convert here"
- "Security advisory: rotate your local Bao seal" (high-priority pin)

Server-authored fields per announcement:

| Field | Type | Notes |
|---|---|---|
| `id` | uuid | content addressable |
| `published_at` | ts | rendering order |
| `expires_at` | ts | hard hide after this |
| `priority` | int | 0 normal, 10 pinned, 100 modal |
| `body_markdown` | text | rendered server-side to HTML, sanitised |
| `cta_label` | string | optional |
| `cta_url` | string | optional, allowlisted to `https://panel.podmaker.sh/*` and `https://podmaker.sh/*` |
| `target_membership_ids` | uuid[] \| `*` | per-membership / global |
| `target_install_versions` | string \| `*` | semver range filter |
| `target_license_tier` | `oss \| trial \| premium \| enterprise \| *` | filter |
| `signature` | base64 | Ed25519 sig over canonical_json of the row |

Server-side targeting happens in `license-api` — the CE only sees the
already-filtered set, so a curious operator can't enumerate other
campaigns by reading their own cache.

### Authoring UI

Lives in the panel (`panel-licensing-ui-plan.md` extension; tracked in
that doc's §6.x backlog after this plan lands). Filament resource
`AnnouncementResource` with the fields above + a live-preview blade
render. Admin-only.

## 7. Key rotation

CE install key rotation:
1. Operator triggers `pdctl ce rotate-key` (or it happens automatically
   on first boot after a `cp_version` major jump).
2. CE generates a new keypair, signs a `key_rotation` envelope with
   the *old* private key, includes the new public key.
3. `license-api` verifies the old signature, updates `public_key`,
   records the rotation in `ce_install_audit`.
4. Server response is signed with the server key for the new public
   key; CE replaces `install.key`.

Server key rotation:
1. New server keypair is registered in the trust ledger (parent §0
   manifestsig touchpoint pattern).
2. Heartbeat response includes `next_server_pubkey` once the rotation
   window opens.
3. CE persists it alongside the current key for the rotation window
   and accepts signatures from either; flips after the cutover ts.

## 8. License + entitlement delivery

When the operator buys a licence via `panel.podmaker.sh`:

1. Panel calls `license-api`'s `POST /v1/license/issue`
   (license-api-plan §3) with `install_uuid = <claimed install>`.
2. `license-api` mints the JWT, attaches it to the next heartbeat
   response for that install.
3. CE receives the JWT in §5 step 5, verifies, persists to
   `storage/app/license.jwt`, flips the relevant `RequiresPremium`
   middleware bindings.
4. Dashboard reflects the new entitlement without a panel restart.

Revoke + downgrade flow uses the same channel — the next heartbeat
returns `{ license: null }` (or `{ license_revoked: true, grace_until }`)
and CE applies the grace logic from parent §6.

## 9. Endpoints (new on `license-api`)

| Method | Path | Min role | Purpose |
|---|---|---|---|
| POST | `/v1/ce/installs/register` | any | install-time registration (signed) |
| POST | `/v1/ce/installs/<uuid>/heartbeat` | install (sig) | hourly heartbeat + pull |
| POST | `/v1/ce/installs/<uuid>/claim` | membership session | claim from panel |
| POST | `/v1/ce/installs/claim-by-code` | install (sig) | claim from CE with one-time code |
| POST | `/v1/ce/installs/<uuid>/rotate-key` | install (sig, old key) | rotate install key |
| GET  | `/v1/ce/installs/<uuid>` | install (sig) \| membership session | install detail |
| POST | `/v1/ce/announcements` | admin | create announcement |
| GET  | `/v1/ce/announcements` | admin | list announcements |

`license-api-plan.md` §3 needs a follow-up patch to declare these.
Tracked there as "pairing endpoints to be merged from
`podmaker-ce-pairing-plan.md` §9".

## 10. Open questions

1. UUIDv7 vs UUIDv4 for `install_uuid` — v7 leaks install ordering but
   is sortable. Acceptable?
2. OTT (one-time code) format — 8-char base32 (case-insensitive,
   human-typable) vs 12-char base58?
3. Heartbeat cadence — 1h fixed vs adaptive (5min during active
   announcement campaigns)? Adaptive needs server push intent
   delivered on previous heartbeat.
4. Should the CE wizard refuse to finish until paired, or always
   allow offline finish? Plan says allow offline; confirm.
5. Per-install RBAC on the membership side — does every membership
   user see every install, or only ones they registered? Affects
   team accounts.
6. Announcement CTA URL allowlist — only `*.podmaker.sh`, or also
   marketing partners?
7. How long after `registration_state = pending_claim` with no claim
   does the row move to `abandoned`? Plan suggests 90 days.

## 11. Out of scope

- Remote command execution / config push to the CE box.
- Webhook delivery from SaaS → CE (NAT-unfriendly; we pull only).
- Multi-account install sharing (one install = one membership at a time).
- Public REST API for third-party install inventory tools.
