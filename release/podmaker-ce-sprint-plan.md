# PodMaker CE — Sprint Plan

> **Status:** active (2026-06-02). Sprint 0 in flight.
> **Owner:** CE working group.
> **Parent:** [`self-hosted-licensing-plan.md`](./self-hosted-licensing-plan.md) §0a.
> **Sibling plans:** [`podmaker-ce-mirror-plan.md`](./podmaker-ce-mirror-plan.md), [`podmaker-ce-install-plan.md`](./podmaker-ce-install-plan.md), [`podmaker-ce-pairing-plan.md`](./podmaker-ce-pairing-plan.md), [`license-api-plan.md`](./license-api-plan.md), [`panel-licensing-ui-plan.md`](./panel-licensing-ui-plan.md).

The sibling plans describe the target architecture. This doc is the
build queue — what ships when, what depends on what, and the
acceptance test for each sprint.

## Sprint shape

- Each sprint is a thin vertical slice that compiles, has at least one
  feature test, and either ships behind a feature flag or behind the
  `RequiresPremium` middleware.
- Sprints land as separate PRs on `main`. CE mirror workflow
  republishes after each merge — no special branch dance.
- A sprint is **done** when the acceptance test passes locally and in
  CI. Polish + hardening are explicit follow-up tickets, not "we'll
  fix it in the next sprint".

## Sprint 0 — Identity primitives (CP-side) — in flight

| Output | File |
|---|---|
| `install_identities` migration (single-row table) | `apps/control-plane/database/migrations/2026_06_02_*` |
| `App\Models\InstallIdentity` model | `apps/control-plane/app/Models/InstallIdentity.php` |
| `App\Services\Pairing\InstallIdentity` service (gen UUIDv7 + Ed25519 keypair, load/persist, never overwrite) | `apps/control-plane/app/Services/Pairing/InstallIdentity.php` |
| `php artisan podmaker:install:identity:ensure` artisan command | `apps/control-plane/app/Console/Commands/EnsureInstallIdentityCommand.php` |
| Feature test: idempotent ensure + signing roundtrip | `apps/control-plane/tests/Feature/Pairing/InstallIdentityTest.php` |

**Acceptance:** `php artisan podmaker:install:identity:ensure` on a fresh
DB generates a UUIDv7, an Ed25519 keypair, persists once; rerun is a
no-op; the service can sign + verify a canonical-json envelope using
the persisted keys.

## Sprint 1 — `license-api` skeleton + register handler

| Output | File |
|---|---|
| New Go module `apps/license-api/` (orchestrator template) | `apps/license-api/{go.mod, cmd/license-api/main.go, internal/{api,ce,keys,db}/}` |
| `go.work` use entry | `go.work` |
| Postgres migrations: `ce_installs`, `ce_install_audit` | `apps/license-api/db/migrations/` |
| Ed25519 server keypair load (env-injected for dev) | `apps/license-api/internal/keys/` |
| `POST /v1/ce/installs/register` handler with signature verify + upsert | `apps/license-api/internal/ce/register.go` |
| Integration test against a dockerised pg | `apps/license-api/internal/ce/register_test.go` |

**Acceptance:** A signed register envelope from Sprint 0's
`InstallIdentity` lands a row in `ce_installs` and returns a signed
`{ registration_id, server_pubkey, announcements_topic }` response that
the CP service verifies. The handler refuses unsigned envelopes (`401`)
and stale envelopes (`>5min ts skew → 401`).

## Sprint 2 — Pairing client + first-boot wiring

| Output | File |
|---|---|
| `App\Services\Pairing\PairingClient` (POST register, persist server pubkey + registration_id) | `apps/control-plane/app/Services/Pairing/PairingClient.php` |
| `config/podmaker.php` block: `pairing.api_base`, `pairing.enabled` | `apps/control-plane/config/podmaker.php` |
| `php artisan podmaker:pair` (idempotent: skip if already paired) | `apps/control-plane/app/Console/Commands/PairCommand.php` |
| Filament `Pages\Setup\Welcome` showing install UUID + pairing status | `apps/control-plane/app/Filament/Admin/Pages/Setup/Welcome.php` |
| Feature test with `Http::fake()` covering paired + offline paths | `apps/control-plane/tests/Feature/Pairing/PairingClientTest.php` |

**Acceptance:** `php artisan podmaker:pair` against a stubbed
`api.podmaker.sh/v1/ce/installs/register` ends with
`install_pair` persisted in the DB and the Filament Welcome page
showing "paired as install_<uuid>". With `pairing.enabled=false` the
command exits 0 without a network call.

## Sprint 3 — Heartbeat (e2e)

| Output | File |
|---|---|
| `POST /v1/ce/installs/<uuid>/heartbeat` handler (sig-verified, updates `last_heartbeat_at`, returns empty announcements/license) | `apps/license-api/internal/ce/heartbeat.go` |
| `App\Services\Pairing\HeartbeatRunner` | `apps/control-plane/app/Services/Pairing/HeartbeatRunner.php` |
| Scheduled task hourly in `app/Console/Kernel.php` | `apps/control-plane/app/Console/Kernel.php` |
| `pdctl ce status` Go subcommand showing local + paired state | `cmd/podmakerctl/ce.go` + `cmd/podmakerctl/ce_status.go` |
| Telemetry counters: `pairing.heartbeat.ok / fail / sig_fail` (Prometheus) | `apps/control-plane/app/Support/Metrics/` |
| Feature + integration tests | both sides |

**Acceptance:** From a checked-out CP + a locally-run license-api
(both pointing at the same dev pg), one register + one heartbeat
round-trip; the SaaS-side `ce_installs.last_heartbeat_at` updates;
`pdctl ce status` reports "paired, last heartbeat <Δ>s ago".

## Sprint 4 — Announcements channel

| Output | File |
|---|---|
| `ce_announcements` + `ce_announcement_deliveries` migrations | `apps/license-api/db/migrations/` |
| Announcement signing on save | `apps/license-api/internal/ce/announcements.go` |
| Heartbeat embeds active announcements filtered by membership + version + tier + last-seen cursor | `apps/license-api/internal/ce/heartbeat.go` (extend) |
| CP cache table `cached_announcements` | migration |
| `<x-pairing.announcements />` Blade component + Filament dashboard widget | `apps/control-plane/resources/views/components/pairing/announcements.blade.php` + `app/Filament/Admin/Widgets/AnnouncementsWidget.php` |
| Panel `AnnouncementResource` (admin authoring + preview) | `apps/control-plane/app/Filament/Admin/Resources/AnnouncementResource.php` |
| Delivery ack on next heartbeat → `ce_announcement_deliveries` rows | both |

**Acceptance:** Admin publishes an announcement targeting tier=`oss`;
within one heartbeat the CE panel renders it on the dashboard;
clicking the CTA records a `clicked` delivery row server-side.

## Sprint 5 — Installer + setup wizard

| Output | File |
|---|---|
| `scripts/installer/ce-install.sh` (Ubuntu 24.04 first, idempotent) | new |
| `deploy/ce/systemd/` unit files (cp, caddy, controllers, bao, pairing.timer) | new |
| `deploy/ce/Caddyfile` (auto-TLS) | new |
| Filament `Pages\Setup\{Welcome,AdminAccount,DomainConfirm,Pair,Finish}` flow | extend |
| `/var/lib/podmaker/first-boot.flag` consume + delete logic | CP middleware |
| Smoke test: dockerised "VPS" → curl-pipe-sh → setup wizard reachable on `:443` | `tests/installer/` |

**Acceptance:** A fresh Ubuntu 24.04 docker container running
`PODMAKER_DOMAIN=test.nip.io PODMAKER_YES=1 sh ce-install.sh` ends with
a Caddy HTTPS endpoint serving the setup wizard, completing the
wizard yields an admin user + paired install_uuid, and the dashboard
shows the test announcement published in Sprint 4's e2e fixture.

## Sprint 6 — License issue + delivery

| Output | File |
|---|---|
| `POST /v1/license/issue` minting JWT bound to `install_uuid` | `apps/license-api/internal/license/issue.go` |
| Heartbeat response embeds new license JWT when claimed install gets one | `apps/license-api/internal/ce/heartbeat.go` (extend) |
| CP-side `App\Services\Licensing\LicenseStore` persists JWT, exposes `allows(string $feature): bool` | new |
| Embedded license public key in `packages/shared-go/pkg/manifestsig/trusted.go` | extend trust ledger |
| `App\Http\Middleware\RequiresPremium` returns `402` with upgrade URL | new |
| One example premium controller method gated behind `RequiresPremium` | choose a no-op for now |
| Panel-side `/installs/<uuid>/issue-license` button calling license-api | extend |

**Acceptance:** Admin clicks "Issue Premium licence" on a paired
install; within one heartbeat the CE panel's `LicenseStore::allows('demo-feature')`
returns true; before issue, the gated controller returns 402 with a
`Link: <upgrade_url>; rel="upgrade"` header.

## Sprint 7+ (backlog, not committed yet)

- CE migration story for existing OSS installs (no install_uuid yet).
- Trial flow `POST /v1/license/trial`.
- First real premium feature: `ai-topology-planner`.
- Key rotation runbook + automation.
- Sharing-detection alerts on the panel.
- pdctl `ce attach-vps` (multi-VPS fleet).

## Dependency graph

```
Sprint 0 (identity)
   │
   ▼
Sprint 1 (license-api register) ──► Sprint 2 (CP pair) ──► Sprint 3 (heartbeat)
                                                                │
                                                                ├─► Sprint 4 (announcements)
                                                                │
                                                                └─► Sprint 6 (license delivery)
                                                                          ▲
Sprint 5 (installer) ─────────────────────────────────────────────────────┘
```

Sprint 5 can land in parallel with Sprint 3+4 — the installer doesn't
care which features are wired, only that the panel and the pairing
client both exist.

## Test + CI rules

- Every sprint adds tests in the same PR; no "test follow-up".
- License-api integration tests run against a dockerised Postgres in
  CI under a new workflow `license-api-ci.yml` (Sprint 1 ships this).
- CP feature tests run under the existing `pr.yml` workflow with
  `Http::fake()` for all `api.podmaker.sh` calls — no live network
  in CI.
- Strip pipeline (`scripts/release/ce-strip.sh --dry-run`) runs on
  every PR that touches `apps/`, `packages/`, `cmd/`, or
  `scripts/release/ce-*`. Leak guard failure blocks the PR.

## Acceptance signal

After Sprint 6 the system has:
- A CE install that registers, pairs, heartbeats, receives
  announcements, receives a license JWT on purchase, and gates one
  premium method end-to-end. Everything from here is feature breadth
  (more premium catalogue, real installer hardening, trial UX).
