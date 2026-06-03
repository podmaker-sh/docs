# Panel Licensing UI — Implementation Plan

> **Status:** draft (2026-06-02). Not started.
> **Owner:** TBD (panel / Filament).
> **Parent:** [`self-hosted-licensing-plan.md`](./self-hosted-licensing-plan.md) §9.Q2 + §9.Q3.
> **Sibling plans:** [`podmaker-ce-mirror-plan.md`](./podmaker-ce-mirror-plan.md), [`podmaker-ce-install-plan.md`](./podmaker-ce-install-plan.md), [`podmaker-ce-pairing-plan.md`](./podmaker-ce-pairing-plan.md), [`license-api-plan.md`](./license-api-plan.md).

The panel at `panel.podmaker.sh` is the customer-facing + admin-facing UI on top of
`license-api`. This doc covers the Filament resources, the customer self-service flow
(download CE, claim license, manage installs), and the admin operations
(issue / revoke / view phone-home).

The panel itself is OSS (it ships in the CE mirror — operators can run their own
panel) but **the license issuance + revocation buttons only work when the panel is
configured against `api.podmaker.sh` with admin credentials**. Self-hosted panels
talk to a no-op stub; only the SaaS panel mints real licenses.

## 0. Kickoff prompt (new session)

> Read `docs/release/panel-licensing-ui-plan.md` and implement §2 (Filament
> resources) + §3.1 (download CE landing page) of the panel. Models live under
> `apps/control-plane/app/Models/Licensing/`. Resources under
> `apps/control-plane/app/Filament/Resources/Licensing/`. Wire the License
> resource's "Issue" action to call `LicenseApiClient::issue(...)` (defined in
> `license-api-plan.md` §3). Stub the client with `Http::fake()` for tests.
> Trial flow (§4) and migration (§5) are follow-up PRs. Open questions in §10
> are blocking.

## 1. Goals + non-goals

| Goal | Why |
|---|---|
| Admin can issue / revoke a license in two clicks | Manual issuance is the only path until trial flow lands |
| Customer can see their installs + premium usage | Reduces support load, surfaces overuse before billing surprise |
| Self-service CE download + premium upgrade CTA | Marketing funnel: stars on CE → trial → paid |
| Phone-home telemetry visible per-install | Detect sharing, debug grace-banner reports |

| Non-goal | Reason |
|---|---|
| License minting on self-hosted panels | Closed to SaaS only |
| Customer-side audit log export | Premium feature itself (`audit-export`), not a panel concern |
| Multi-tenant license sharing UX | Parent §10 out-of-scope rules this out |

## 2. Filament resources

All under `apps/control-plane/app/Filament/Resources/Licensing/`.
Models under `apps/control-plane/app/Models/Licensing/`.

### 2.1 `LicenseResource`  *(admin-only)*

| Column | Source |
|---|---|
| `license_id` | `licenses.id` |
| `customer` | `customers.name` |
| `tier` | `licenses.tier` |
| `features` | `licenses.features` (chip list) |
| `seats` | `licenses.seats` |
| `issued_at` | `licenses.created_at` |
| `expires_at` | `licenses.exp` |
| `revoked_at` | `licenses.revoked_at` (red badge if set) |
| `last_seen` | latest `phone_home.seen_at` (relation) |

Actions:
- **Issue** (header action) — modal with customer picker, tier select, feature
  multiselect (driven by `config/premium.php`), seats integer, expiry datetime.
  On submit → `LicenseApiClient::issue(...)`.
- **Revoke** (row action) — confirmation modal with reason textarea →
  `LicenseApiClient::revoke($id, $reason)`.
- **View phone-home** (row action) — relation manager listing last 90 days of
  `phone_home` rows for this `install_uuid`.

### 2.2 `InstallResource`  *(read-only, both admin + customer)*

One row per `install_uuid` ever observed by phone-home.

| Column | Source |
|---|---|
| `install_uuid` | `installs.uuid` |
| `customer` | `installs.customer_id` |
| `cp_version` | last `phone_home.cp_version` |
| `first_seen` | `installs.created_at` |
| `last_seen` | latest `phone_home.seen_at` |
| `current_license` | `installs.license_id` (relation badge) |
| `fingerprint_status` | `installs.fingerprint_match_streak` (green ≥7d, amber < 7d) |
| `ip_distinct_count_24h` | aggregate over `phone_home` (amber if > 3) |

Customer scope (`InstallPolicy`): customers see only their own installs;
admins see all.

### 2.3 `PremiumCallResource`  *(read-only, admin + customer)*

One row per `premium_calls` audit row from `license-api`. Filters:
`install_uuid`, `feature`, `date_range`, `status`. Drives the customer's
"usage this month" dashboard.

### 2.4 `CeInstallResource`  *(read-only, admin + customer)*

One row per CE install paired with `panel.podmaker.sh` via the channel
in [`podmaker-ce-pairing-plan.md`](./podmaker-ce-pairing-plan.md) §3.

| Column | Source |
|---|---|
| `install_uuid` | `ce_installs.install_uuid` |
| `membership` | `ce_installs.membership_id` |
| `cp_version` | last heartbeat |
| `channel` | `ce.env` channel (`stable` / `edge`) |
| `host_fingerprint` | `ce_installs.host_fingerprint` |
| `registered_at` | `ce_installs.created_at` |
| `last_heartbeat_at` | `ce_installs.last_heartbeat_at` |
| `registration_state` | `pending_claim` / `paired` / `abandoned` |
| `attached_vps_count` | last heartbeat |
| `sites_count` | last heartbeat |
| `current_license` | relation to `licenses` (issue button if null) |

Actions:
- **Issue licence for this install** — pre-fills the License issue
  modal with `install_uuid`, surfaces seat-vs-feature picker, delivery
  is automatic via the next heartbeat.
- **Send announcement to this install** — quick-compose targeting one
  install instead of a whole membership.
- **Abandon** — admin only; flips `registration_state = abandoned`,
  forces re-pairing on next heartbeat.

Customer scope (`CeInstallPolicy`): customer sees only own
membership's installs.

### 2.5 `AnnouncementResource`  *(admin-only)*

Remote-managed dashboard widget content (parent §0a + pairing plan §6).

| Column | Source |
|---|---|
| `id` | `announcements.id` |
| `title` | `announcements.title` |
| `body_markdown` | full markdown (excerpt in table) |
| `cta_label` / `cta_url` | optional |
| `priority` | 0 / 10 / 100 |
| `target_membership_ids` | tag list (`*` = global) |
| `target_install_versions` | semver range (`*` = any) |
| `target_license_tier` | `oss` / `trial` / `premium` / `enterprise` / `*` |
| `published_at` / `expires_at` | scheduling window |
| `signature` | computed on save (Ed25519 over canonical_json) |

Actions:
- **Publish** — sets `published_at`, signs the row with the server
  key, makes it eligible for the next CE heartbeat pull.
- **Preview as install** — renders the Blade widget against a chosen
  CE install's filter set so admin sees exactly what the operator
  will see.
- **Pull stats** — read-only metrics from `ce_announcement_deliveries`
  (sent / acked / clicked).

### 2.6 `LicenseEventResource`  *(read-only, admin)*

Append-only log of `license.issued`, `license.revoked`,
`license.suspected_sharing` webhook deliveries (parent §7) for forensics.

## 3. Customer self-service

### 3.0 CE membership home (`/installs`)

The new CE-aware customer surface. Every membership account gets:

- A list of their paired CE installs (`CeInstallResource` filtered).
- A "Pair a new CE install" button → shows the OTT one-time-code flow
  from pairing plan §4.2.
- A "Buy / activate licence" panel: tier picker → checkout → on
  payment success, mints licence via `LicenseApiClient::issue($install_uuid)`
  → next heartbeat pushes the JWT to the install (pairing plan §8).
- "What's included in my licence" — feature list resolved from
  `licenses.features`, cross-referenced against the premium catalogue
  in parent §3.
- "Announcements going to my installs" — read-only list of currently
  active `AnnouncementResource` rows the operator's installs will
  receive (transparency — no secret targeting against the operator).

### 3.1 CE download + premium CTA landing

Public page at `panel.podmaker.sh/download/ce`:

- Curl one-liner to install the CE binary (points at the GitHub release asset
  on `podmaker-sh/podmaker-ce`, see `podmaker-ce-mirror-plan.md` §9.Q1).
- "Want premium?" CTA → checkout flow → on payment success, mint license via
  `LicenseApiClient::issue()` and email the JWT + install instructions.

### 3.2 Customer dashboard at `/account/installs`

- List of installs (filtered `InstallResource` view).
- For each install: "Download license JWT" button (re-fetches from
  `license-api` — never stores plaintext in panel DB).
- "Premium usage this month" card from `PremiumCallResource`.
- "Upgrade to Enterprise" CTA if currently on Premium.

### 3.3 Customer self-revoke

A customer can revoke their own license JWT (e.g. machine stolen) without
opening a support ticket. Action calls `LicenseApiClient::revoke()` with
`reason: 'customer_initiated'`. Admin gets a Filament notification.

## 4. Trial flow

(Resolves parent §9.Q6.)

1. CE-installed CP boots, no license JWT present.
2. CP `LicenseService::load()` calls
   `POST https://api.podmaker.sh/v1/license/trial` with `{ install_uuid,
   fingerprint, cp_version }`.
3. `license-api` checks `installs` table — if `install_uuid` already had a
   trial, returns 409. Else mints a 14-day Premium JWT (all `features: ['*']`)
   and returns it. Logs `trial.minted` panel webhook.
4. CP caches the trial JWT under `storage/app/license.jwt`, banner shows
   "Premium trial: 14 days remaining".
5. On day 12, banner becomes "Trial ending in 2 days — upgrade now".
6. On expiry, premium endpoints flip to 402 with the standard `RequiresPremium`
   middleware path.

Panel UX:
- Trial installs appear in `InstallResource` with a `trial` badge.
- Admin can extend (one extra 7-day grace) via action — calls
  `LicenseApiClient::extend_trial(...)`.

## 5. Migration (existing OSS users)

Parent §9.Q3 — what happens to current self-hosters when they pay?

1. Customer signs up + pays via panel.
2. Panel asks for `install_uuid` (visible via `pdctl license status` on the CE
   binary — that subcommand is in CE because it can also report "no license").
3. `LicenseApiClient::issue(install_uuid, tier, features, seats, exp)`.
4. Email customer the JWT + the `pdctl license install --token <jwt>` command.
5. Customer runs it on the CP host; phone-home flips premium ON.

For customers without `install_uuid` yet (clean install during signup), the
trial flow (§4) bootstraps the `install_uuid` on first boot — they upgrade by
purchasing, which finds the existing trial install and converts it.

## 6. Telemetry dashboards

Filament widgets on the admin home:

- **Installs by tier** (stacked bar, last 30d active).
- **Premium calls by feature** (line, last 7d).
- **Phone-home reachability** (% of expected pings received in last 24h).
- **Sharing alerts** (count of `license.suspected_sharing` events last 7d).

All sourced from `license-api` via a thin `LicenseApiClient::telemetry()`
read-only endpoint (add to `license-api-plan.md` §3 in a follow-up).

## 7. Auth + ACL

- Admin role on `panel.podmaker.sh` → can issue/revoke any license, view all
  installs.
- Customer role → scoped to their `customer_id` — see own installs +
  premium-calls + license JWTs only.
- Self-hosted panels (CE) — License/Install resources hidden behind a feature
  flag `panel.licensing_admin` defaulting to `false`. The OSS panel still
  shows the customer-side `/account/installs` view but the issue/revoke actions
  no-op with a "managed at panel.podmaker.sh" banner.

## 8. Notifications

- License expiry: 14d / 7d / 1d email + in-panel notification.
- License revoked: immediate email with reason.
- Grace countdown: panel banner at < 72h reachability gap, email at < 24h.
- Sharing alert: admin Filament notification + Slack webhook to internal
  `#alerts-licensing`.

## 9. Migration of existing tier YAMLs

T5 Enterprise tier YAMLs (`tiers/t5-enterprise-{aws,gcp,multi-cloud}.yaml`)
already carry `metadata.license_tier` + `metadata.license_features` per parent
§0 touchpoints. The panel's `TierPromoteController` (parent touchpoint row) needs
a new pre-flight check:

```php
if ($tier->metadata->license_tier !== 'oss'
    && !app(LicenseService::class)->allows($tier->metadata->license_features)) {
    throw new PremiumRequiredException($tier->metadata->license_features);
}
```

This is the single chokepoint where premium tier promotion is gated. Lives in
`apps/control-plane/app/Http/Controllers/Api/TierPromoteController.php`.

## 10. Open questions

1. Filament v3 or stay on the panel's current Filament version? Affects
   resource API.
2. Where does the panel-side `LicenseApiClient` live — `app/Services/Licensing/`
   or `packages/php-license-client/`? If the latter, the OSS CE binary can
   reuse it without owning the SaaS-only minting path.
3. Customer self-revoke (§3.3) — require 2FA confirmation? Lost-machine threat
   model says yes.
4. Trial flow IP rate limiting — one trial per `install_uuid` + one per source
   `/24` per 30 days? Need to balance abuse vs legitimate operator running
   multiple environments.
5. Do we expose phone-home telemetry to customers (read-only), or admin-only?
   Customers seeing their own pings is debugging-friendly but adds support
   surface.

## 11. Out of scope

- Stripe / billing integration (separate doc).
- Customer-side audit log export UI (lives in `audit-export` premium feature).
- Multi-tenant license assignment UX.
- Email templating refactor (use existing panel mail layer).
