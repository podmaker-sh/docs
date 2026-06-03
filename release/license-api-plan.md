# `license-api` Service — Implementation Plan

> **Status:** draft (2026-06-02). Not started.
> **Owner:** TBD (backend / platform).
> **Parent:** [`self-hosted-licensing-plan.md`](./self-hosted-licensing-plan.md) §4–§7.
> **Sibling plans:** [`podmaker-ce-mirror-plan.md`](./podmaker-ce-mirror-plan.md), [`podmaker-ce-install-plan.md`](./podmaker-ce-install-plan.md), [`podmaker-ce-pairing-plan.md`](./podmaker-ce-pairing-plan.md), [`panel-licensing-ui-plan.md`](./panel-licensing-ui-plan.md).

This is the closed-source service that lives at `api.podmaker.sh`. It does three things:

1. **Issue / verify / revoke** Ed25519-signed license JWTs.
2. **Serve the premium feature endpoints** (`/v1/premium/<feature>/*`) — the actual
   server-side work the self-hosted CP delegates to.
3. **Receive phone-home pings** from CPs and emit telemetry to the panel.

It does *not* host any UI — that is the panel's job
([`panel-licensing-ui-plan.md`](./panel-licensing-ui-plan.md)).

## 0. Kickoff prompt (new session)

> Read `docs/release/license-api-plan.md` and implement §3 (issue + verify only,
> skip revoke for now) and §5 (Ed25519 key handling) of a new Go service under
> `apps/license-api/`. Follow the `apps/orchestrator` layout for cmd/internal
> split, database wiring, and config. Reuse `packages/shared-go/pkg/manifestsig`
> for the Ed25519 verify path. Stub the premium endpoints in §4 with a single
> handler that returns 501 — those land in a follow-up PR per feature. Open
> questions in §10 are blocking.

## 1. Goals + non-goals

| Goal | Why |
|---|---|
| Single source of truth for premium entitlements | Revoke-on-demand is the entire pitch over signed-license phone-home (§1 parent doc) |
| Stateless verify (Ed25519 sig check) + stateful revoke (DB lookup) | CP can verify offline; revoke needs phone-home |
| Premium feature endpoints terminate here, not the CP | Server-side compute = unhackable (§1 parent doc) |
| Phone-home telemetry feeds panel dashboards | Detect license sharing, abuse, abnormal feature usage |

| Non-goal | Reason |
|---|---|
| Customer-facing UI | Panel concern |
| OSS distribution | Stays closed-source on `podmaker-sh/license-api` private repo |
| Air-gap license issuance | Separate runbook (Enterprise Air-Gap SKU) |

## 2. Service shape

- **Language:** Go (reuse `packages/shared-go/pkg/manifestsig`).
- **Layout:** mirrors `apps/orchestrator/`:
  ```
  apps/license-api/
    cmd/license-api/main.go
    internal/
      issue/      # /v1/license/issue
      verify/     # /v1/license/verify
      revoke/     # /v1/license/revoke
      premium/    # /v1/premium/<feature>/*
      keys/       # Ed25519 key load + rotation
      phonehome/  # /v1/license/phone-home (cron target)
      telemetry/  # emits to panel webhook
    config/
    db/migrations/
    deploy/
  ```
- **Hosting:** multi-region (us-east-1 + eu-west-1) behind `api.podmaker.sh`.
  PgBouncer → managed Postgres for license + install + audit tables.
- **TLS:** terminated at the LB; service speaks plaintext internally.
  CP-side cert pin (parent §7) enforces `*.podmaker.sh` only.

## 3. Endpoints — license lifecycle

### `POST /v1/license/issue`  *(min role: admin via panel)*
Request:
```json
{
  "install_uuid": "...",
  "tier": "premium" | "enterprise" | "enterprise-airgap",
  "features": ["ai-topology-planner", "anomaly-detector"],
  "seats": 25,
  "exp_unix": 1735689600,
  "customer_id": "..."
}
```
Response: `{ "license_jwt": "...", "license_id": "lic_<uuid>" }`.
Side effects: writes `licenses` + `license_issuances` rows; emits panel webhook
`license.issued`.

### `POST /v1/license/verify`  *(min role: any — CP boot + 24h cron)*
Request:
```json
{ "install_uuid": "...", "fingerprint": "sha256(...)", "license_hash": "..." }
```
Response:
```json
{
  "revoked": false,
  "tier": "premium",
  "features": ["ai-topology-planner"],
  "grace_until": 1735689600,
  "license_hash": "...",
  "rotated_jwt": null
}
```
Logs `phone_home` row with timestamp + IP + fingerprint. Detects fingerprint
mismatch (license JWT used on a different install) and returns
`{ "revoked": true, "reason": "fingerprint_mismatch" }`.

### `POST /v1/license/revoke`  *(min role: admin via panel)*
Request: `{ "license_id": "...", "reason": "..." }`. Sets `licenses.revoked_at`,
emits panel webhook `license.revoked`. Next CP phone-home returns
`{revoked: true}`; CP banner counts down the 7-day grace.

### `GET /v1/license/{license_id}`  *(min role: admin)*
Returns issuance + revoke + last 90 days of phone-home rows.

### CE pairing + announcement endpoints

The CE distribution adds a parallel `/v1/ce/*` namespace covering
install registration, heartbeat, membership claim, key rotation, and
the remote-managed announcement channel. The full spec lives in
[`podmaker-ce-pairing-plan.md`](./podmaker-ce-pairing-plan.md) §9 —
this service hosts the handlers under
`apps/license-api/internal/ce/`. License JWTs minted via
`/v1/license/issue` for a `claimed` install are delivered through the
heartbeat response (pairing plan §8), not via the legacy
`license.jwt` paste flow.

## 4. Endpoints — premium feature proxy

Each premium feature gets its own subtree. Common middleware:

1. Authorise `Authorization: Bearer <license_jwt>` against `licenses` table.
2. Reject if revoked, expired, or feature not in `features`.
3. Rate-limit per `install_uuid` (per-feature limits in `config/`).
4. Audit-log to `premium_calls` (install, feature, latency, status).

### `POST /v1/premium/ai-topology-planner/plan`
Body: `{ interview: [...], hints: {...} }`. Returns:
`{ tier_yaml: "...", reasoning: "...", confidence: 0.92 }`.
Backed by an LLM call — see §8 for the pilot.

### `POST /v1/premium/anomaly-detector/scan`
Body: `{ metric_window: {...} }`. Returns: `{ anomalies: [...] }`.

### `POST /v1/premium/multi-region-scheduler/plan`
Body: `{ regions: [...], constraints: {...} }`. Returns scheduling decisions.

### `POST /v1/premium/saml-sso/init` + `/callback`  *(Enterprise tier)*
Standard SAML AuthnRequest / AuthnResponse exchange.

### `POST /v1/premium/audit-export/dispatch`  *(Enterprise tier)*
Signs + forwards audit events to the customer's configured SIEM webhook.

> **First-feature pilot:** ship `ai-topology-planner` end-to-end before
> implementing the others (§8). The pipe is the risky part — the second
> feature copies the first.

## 5. Ed25519 key handling

Mirror the `manifestsig` ledger pattern exactly (parent §0 touchpoint row).

- **Storage:** private key in `apps/license-api/keys/license-ed25519.key`
  (sealed via SOPS, KMS-wrapped). Loaded into memory on boot only — never
  written to disk in plaintext.
- **Public key:** embedded in every CP binary at build time via
  `packages/shared-go/pkg/manifestsig/trusted.go` (extend the trust ledger
  with a `license_keys[]` block). Same rotation story as manifestsig.
- **Rotation runbook:** write `docs/release/license-signing-rotation.md` as the
  procedural twin of `manifest-signing-rotation.md` once the first key is in
  production. Key rolls quarterly + on incident.
- **JWT shape:** see parent doc §4. `kid` header field selects the active
  public key from the trust ledger.

## 6. Phone-home protocol

```
CP cron (24h) ──► POST /v1/license/verify
                       ├─ stateless: sig + exp + fingerprint
                       ├─ stateful:  revoke flag in DB
                       └─ logs row in phone_home
                  response includes grace_until
CP behaviour:
  reachable + revoked=false → premium ON
  reachable + revoked=true  → premium OFF after grace_until
  unreachable               → 72h soft grace → 7d hard grace → premium OFF
```

`phone_home` table fields: `install_uuid`, `seen_at`, `ip`, `fingerprint_match`,
`license_hash`, `cp_version`. Used by panel telemetry dashboards.

## 7. Anti-bypass (service side)

- mTLS optional, cert pin mandatory on CP outbound (parent §7).
- Per-`install_uuid` rate limits (default 10 req/s, burst 50; configurable per feature).
- Per-`license_id` anomaly detector: if same license phones home from >3 distinct
  IPs in 24h, flag for admin review and emit `license.suspected_sharing` webhook.
- Sign phone-home response with the same Ed25519 key so CP rejects a MITM trying
  to flip `revoked` to false.
- All endpoints behind WAF rules that drop non-`*.podmaker.sh`-CN traffic at the
  client-cert layer (Premium feature endpoints, not the verify path — verify must
  work from any CP regardless of cert).

## 8. First-feature pilot — `ai-topology-planner`

This is the canary that proves the whole pipe works end-to-end:

1. CP `AiPlannerClient` (parent §5) sends interview transcript.
2. `license-api` middleware authorises, rate-limits, logs.
3. Handler calls the LLM provider, parses YAML, validates against `tiers/_schema.json`.
4. Returns plan + reasoning + confidence.
5. CP renders in the panel; user accepts → goes through normal `pdctl tier promote`.

Success criteria for cutover:
- p95 latency < 8s end-to-end.
- 0 license-bypass paths discoverable via fuzz test of the CP client.
- Panel sees `premium_calls` rows + `phone_home` rows for the pilot install.

## 9. Deploy + SRE

- Multi-region active/active behind GeoDNS (parent §9.Q5).
- Postgres in primary region, read replicas in secondary; verify path can serve from
  replica (eventually consistent — revoke takes up to replication lag to propagate;
  document as accepted tradeoff).
- SLOs: verify p99 < 200ms, premium p99 per-feature (set after pilot).
- Observability: OTel traces, Prometheus metrics scraped to the existing internal
  Grafana. New dashboards in `panel-licensing-ui-plan.md` §6 read from the same
  Prometheus.
- Backup: standard managed-Postgres PITR; `licenses` table is the only true source
  of truth that can't be regenerated from elsewhere.

## 10. Open questions

1. Which LLM provider for `ai-topology-planner`? Affects vendor lock-in + DR.
2. Do Enterprise customers get a dedicated `api.<customer>.podmaker.sh` shard, or
   share the multi-tenant cluster with per-customer rate limits?
3. Phone-home payload signing key — same `install_uuid` ephemeral key as parent §7,
   or a separate per-install key generated on first boot?
4. Where does the `license-api` private repo live? `podmaker-sh/license-api`
   (new private repo) vs `apps/license-api/` inside this monorepo? Affects who
   has commit access.
5. Trial flow (parent §9.Q6) — does `license-api` mint a 14-day trial JWT on
   first CP boot via an unauthenticated `/v1/license/trial` endpoint, or does the
   panel signup flow mint it?

## 11. Out of scope

- Air-gap Enterprise license bundle generator.
- Customer-facing UI (panel concern).
- Premium feature implementations beyond the `ai-topology-planner` pilot.
- License-key encryption-at-rest beyond what managed Postgres already provides.
