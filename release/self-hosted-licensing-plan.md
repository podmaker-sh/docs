# Self-Hosted Limited Edition + Premium Licensing Plan

> **Status:** draft (2026-06-02). Not started — scaffold in a future session.
> **Owner:** TBD.
> **Decision so far:** Premium features bind to `podmaker.sh` via the
> **server-side premium** pattern (the actual work runs on our infra; the
> self-host CP is a thin proxy). The alternative patterns (signed-license
> phone-home, build-time gates) were considered and de-prioritised — they
> are weaker bindings and offer worse anti-piracy guarantees.
>
> **Phase split:** this doc is the umbrella. The three sibling plans
> below are the actual build queues — pick them up in any order, they
> share only the parent decisions in §1–§4 and §7.
>
> 1. [`podmaker-ce-mirror-plan.md`](./podmaker-ce-mirror-plan.md) — the
>    public CE mirror workflow + strip pipeline. Ships first; unblocks OSS
>    distribution and is the only piece the CE repo itself depends on.
> 2. [`license-api-plan.md`](./license-api-plan.md) — the closed-source
>    `api.podmaker.sh` service that issues + verifies + revokes JWTs and
>    hosts the premium feature endpoints. Pilot feature:
>    `ai-topology-planner`.
> 3. [`panel-licensing-ui-plan.md`](./panel-licensing-ui-plan.md) — the
>    Filament resources, customer self-service, trial flow, telemetry
>    dashboards. Depends on `license-api`'s issue/verify endpoints landing
>    first.
>
> **Suggested cutover order:** CE mirror (done) → CE install + setup
> wizard → CE↔SaaS pairing (register + claim + announcement channel)
> → `license-api` issue+verify → panel License/Install/CeInstall +
> Announcement resources → trial flow → first premium feature
> (`ai-topology-planner`) → remaining premium catalogue (§3) one at a
> time.

## 0. Kickoff prompt (new session)

Paste verbatim into a fresh Claude Code session:

> Read `docs/release/self-hosted-licensing-plan.md` and implement
> §5 (Components to build) in dependency order: `LicenseService` →
> `PremiumClient` → `RequiresPremium` middleware → first
> `AiPlannerClient`. Wire the JWT verify path against the Ed25519
> public key embedded via `manifestsig` (same pattern as the
> kinds-manifest ledger). Honour the runtime behaviour rules in §6.
> Don't touch the `license-api` HTTP service yet — stub
> `https://api.podmaker.sh/premium/*` with `Http::fake()` in the
> CP-side tests for this PR. Open questions in §9 are blocking;
> ask before guessing.

### Touchpoints already wired this session

| What | Where | Note |
|---|---|---|
| Tier YAML schema field `metadata.license_tier` | `tiers/_schema.json` + `packages/shared-go/pkg/tier/spec.go` | OSS / premium / enterprise / enterprise-airgap |
| Tier YAML schema field `metadata.license_features[]` | same | per-feature entitlement |
| Pre-canned T5 enterprise tiers using both fields | `tiers/t5-enterprise-{aws,gcp,multi-cloud}.yaml` | reference shape |
| Cost estimator surcharges for `license_tier` + features | `packages/shared-go/pkg/tier/cost.go` (last branch of `Estimate`) | placeholder bill numbers |
| Ed25519 sign + trust ledger embed pattern | `packages/shared-go/pkg/manifestsig/` | reuse for license JWT verify |
| `manifestsig.VerifyAgainstTrusted` ledger walk | `packages/shared-go/pkg/manifestsig/trusted.go` | copy the rotation story |
| Rotation runbook (procedural twin) | `docs/release/manifest-signing-rotation.md` | mirror sections for license keys |
| pdctl tier promote dispatch path | `cmd/podmakerctl/tier.go` → `apps/control-plane/app/Http/Controllers/Api/TierPromoteController.php` → orchestrator `/v1/self-deploy/promote` | premium gate lives somewhere along this chain |
| Business CI mirror pattern reused for CE | `.github/workflows/publish-vault-bridge-agent.yml`, `.github/workflows/publish-podmaker-bridge.yml` | twin → `publish-podmaker-ce.yml` (§5) drives the OSS mirror with allowlist + strip |

## 0a. CE distribution shape (locked 2026-06-02)

CE is **not** a "trimmed full PodMaker" — it is a deliberately narrow
single-host hosting panel that operators install on one VPS / dedicated
box and grow horizontally by attaching more VPSes through the panel.
Multi-provider orchestration, vault-bridge, and podmaker-bridge are
out-of-scope by design (they are SaaS-only concerns).

| Dimension | CE | Premium / Enterprise (SaaS) |
|---|---|---|
| **Install** | `curl -fsSL https://get.podmaker.sh/ce \| sh` or one-shot ssh exec | SaaS — no install |
| **Bootstrap** | bind to host IP or chosen domain → setup wizard (admin user+pass) → live | n/a |
| **Topology** | one install host + N operator-attached VPSes running `apps/agent` | full multi-cloud orchestration |
| **Vault** | local Bao only (`apps/vault-broker`) | local Bao + customer cloud KMS via `vault-bridge-agent` |
| **Provider scope** | none — the install host + manually-attached VPSes are the entire fleet | AWS / GCP / DO / Hetzner via `cloud-broker` |
| **Shared services** | included (cache, db, lb, metrics controllers, event-bridge) | same controllers + multi-region scheduler (premium) |
| **Site deploy** | included | included + deploy-insights (premium) |
| **License** | optional — CE runs free without any license. Adding a license unlocks premium apps via the standard `RequiresPremium` middleware | always-on, server-side compute via `api.podmaker.sh` |
| **CE↔SaaS link** | Ed25519 keypair generated at install; CE pairs with `panel.podmaker.sh` (membership account) → buy license → JWT injected | n/a |
| **`pdctl` scope** | `pdctl --url https://<ce-host>` against the operator's own CE panel; refuses premium subcommands without a paired license | full pdctl |
| **Caps** | none (no soft limits on apps/sites/attached VPSes) — premium gating is feature-based, not quota-based | n/a |
| **Tier YAML support** | `t1-single-host-*`, plus a new `tce-attached-fleet-*` family generated by `apps/control-plane` from the attached-VPS inventory | T1–T5 |
| **Telemetry** | opt-in only; pairing handshake + license verify pings only when a license is attached | full |
| **Out-of-scope apps** | `apps/cloud-broker`, `apps/vault-bridge-agent`, `apps/podmaker-bridge`, `apps/mesh-controller` | included |
| **Auto-update** | manual `git pull` on the CE clone or `pdctl self-update --channel ce` (pulls from podmaker-sh/podmaker-ce releases) | rolling SaaS deploys |
| **Support** | community (GitHub issues on `podmaker-sh/podmaker-ce`) | paid SLA |
| **Branding** | "Powered by PodMaker" footer required (link to <https://podmaker.sh>); removable with paid licence flag | n/a |

The implementation queue for the CE shape is:

1. [`podmaker-ce-mirror-plan.md`](./podmaker-ce-mirror-plan.md) — the
   public mirror tooling (already implemented).
2. [`podmaker-ce-install-plan.md`](./podmaker-ce-install-plan.md) — the
   `get.podmaker.sh/ce` curl-sh installer, ssh-exec form, and the
   first-boot setup wizard.
3. [`podmaker-ce-pairing-plan.md`](./podmaker-ce-pairing-plan.md) — the
   Ed25519 keypair install pairing, `panel.podmaker.sh` membership
   binding, and the "buy / activate / view-included-apps" flow.

## 1. Why server-side premium

| Pattern | Hackable? | Notes |
|---|---|---|
| **Server-side compute (chosen)** | Effectively no — the premium logic is not on the customer's disk. License revoke = feature dies. | Requires reachability to `*.podmaker.sh`. Air-gap customers stay on OSS or pay for the offline SKU. |
| Signed-license phone-home | Yes — anyone with the private key or a fake license server can re-enable. Code is local. | OK fallback for features that *must* run offline (none today). |
| Build-time feature flags | Trivially — the moment the private repo leaks, everything is unlocked. | Don't use. |

## 2. Edition matrix

| Edition | Distribution | Premium catalogue |
|---|---|---|
| **OSS (self-hosted, free)** | `github.com/podmaker-sh/podmaker-ce` (Community Edition) — a deliberately trimmed mirror of this private business repo. Builds the same OSS binaries but ships without any premium-bound code, the `license-api` HTTP client, or this doc. MIT-equivalent licence. | T1–T3 self-deploy, plan apply/validate, vault-broker, mesh-controller, panel, every OSS controller. No phone-home. No telemetry beyond opt-in. |
| **Premium (paid, self-hosted)** | Premium subscribers pull the closed-source distribution (binary or private-repo access; TBD §9) + a `license.jwt` issued by `panel.podmaker.sh` | Adds the catalogue in §3. Each premium call proxies to `api.podmaker.sh/premium/*`. |
| **Enterprise (paid + seat-based)** | Same closed-source distribution as Premium | Adds SAML / audit export / fleet rollup (also server-side). |
| **Enterprise Air-Gap (paid, bespoke)** | Closed-source binaries + a 1–3 year **offline** license bundle | A *signed bundle* of premium features rolled into the binary at issuance time (pattern #2). Sold per-deal; pricing reflects the loss of revoke-on-demand. |

> **Repo split:** this private repo (`github.com/podmaker-sh/podmaker`) carries
> everything — premium clients, `license-api`, customer-side audit code,
> internal runbooks. The public CE repo (`podmaker-ce`) is generated
> from it by the **business CI mirror pattern** already used for
> `apps/vault-bridge-agent` and `apps/podmaker-bridge`
> (`.github/workflows/publish-vault-bridge-agent.yml`,
> `.github/workflows/publish-podmaker-bridge.yml`): checkout src + dst,
> wipe dst tree except `.git`, copy src subset over, regenerate
> `README.md`, commit + push with a scoped PAT. CE differs in one
> respect — it is not a single `apps/*` dir, so the copy step is
> driven by `scripts/release/ce-allowlist.txt` +
> `scripts/release/ce-denylist.txt` and a `scripts/release/ce-strip.sh`
> guard that runs `grep -r` for premium namespaces and fails the
> workflow on leak. Triggers mirror the bridge workflows: daily cron
> (drift catch), `workflow_dispatch`, push to `main` touching any
> allowlisted path, and `ce-v*` tag for snapshotting releases.
> Closed-source distribution paths for Premium/Enterprise are §9
> open questions.

### 2a. Limited apps subset (what ships in CE)

CE mirrors only the OSS surface. Allowlist excerpt (full list lives
in `scripts/release/ce-allowlist.txt` once authored):

| Included | Path | Notes |
|---|---|---|
| Control plane (OSS profile) | `apps/control-plane/` minus `app/Services/Premium/`, `app/Http/Middleware/RequiresPremium.php` | premium client classes stripped at sync time |
| Vault bridge agent | `apps/vault-bridge-agent/` | already mirrored separately; bundled here for full-stack CE build |
| Podmaker bridge | `apps/podmaker-bridge/` | operator-side loopback daemon |
| Mesh controller | `apps/mesh-controller/` | full |
| Orchestrator (OSS bits) | `apps/orchestrator/` minus closed self-deploy promote paths | strip via denylist patterns |
| Shared Go packages | `packages/shared-go/` minus premium-only feature packages | `pkg/tier`, `pkg/manifestsig`, etc. |
| pdctl (OSS subcommands) | `cmd/podmakerctl/` minus `license.go` | license subcommand is premium-only |
| Tier YAMLs (T1–T3) | `tiers/t1-*.yaml`, `tiers/t2-*.yaml`, `tiers/t3-*.yaml` | T5 enterprise YAMLs denylisted |

Denylist (hard reject — workflow fails if any of these slip through):

- `apps/license-api/**`
- `apps/control-plane/app/Services/Premium/**`
- `apps/control-plane/app/Http/Middleware/RequiresPremium.php`
- `apps/control-plane/config/premium.php`
- `cmd/podmakerctl/license.go`
- `tiers/t5-enterprise-*.yaml`
- `docs/release/self-hosted-licensing-plan.md` (this doc itself)
- Any file matching `grep -lE "namespace.*Premium|license-api|RequiresPremium"`.

## 3. Premium catalogue (initial)

### Tier: Premium
- **ai-topology-planner** — LLM-driven plan generation (interview → YAML).
- **multi-region-scheduler** — Aurora Global + cross-region replication orchestration.
- **anomaly-detector** — metric stream → outlier alerts.
- **support-ticket-router** — SLA timer + priority queue.
- **deploy-insights** — build/deploy analytics dashboards.
- **premium-marketplace** — signed download for premium templates.

### Tier: Enterprise
- **saml-sso** — IdP roundtrip via `auth.podmaker.sh`.
- **audit-export** — signed SIEM webhook router.
- **fleet-telemetry-aggregation** — cross-workspace rollup.

### Tier: OSS (unrestricted)
- Self-deploy (T1–T3), `pdctl` plan lifecycle, the entire controller fleet, vault-broker, mesh-controller, the panel itself.

## 4. License key (JWT)

Ed25519 signed by `license.podmaker.sh`. Public key is **embedded in the CP
binary** so the signature check works offline (the *signature* needs no
phone-home; the *revoke check* does).

```json
{
  "iss": "license.podmaker.sh",
  "sub": "install_<uuid>",
  "tier": "premium",
  "features": ["ai-topology-planner", "anomaly-detector"],
  "seats": 25,
  "exp": 1735689600,
  "iat": 1704067200,
  "fingerprint": "sha256(install_uuid + first_boot_machine_id)"
}
```

`fingerprint` pins the JWT to the install at first boot — copying the JWT to
a second CP causes a mismatch on phone-home and triggers the 7-day grace
banner.

## 5. Components to build

```
apps/license-api/                                # new Go service, panel.podmaker.sh subset
  cmd/license-api/
  internal/issue/        # /v1/license/issue          (admin)
  internal/verify/       # /v1/license/verify         (CP boot + 24h cron)
  internal/revoke/       # /v1/license/revoke         (admin)
  internal/premium/      # /v1/premium/<feature>/...  (actual feature endpoints)

apps/control-plane/
  config/premium.php                              # endpoint map + license cache TTL
  app/Services/Premium/
    License.php                                   # Ed25519 verify + cache (1h)
    PremiumClient.php                             # base HTTP client
    AiPlannerClient.php
    AnomalyDetectorClient.php
    AuditExportClient.php
    ...
  app/Http/Middleware/RequiresPremium.php         # 402 in OSS mode; pass through w/ license
  app/Console/Commands/LicenseStatusCommand.php   # `php artisan license:status`

cmd/podmakerctl/
  license.go                                     # `pdctl license install --token <jwt>`
                                                 # `pdctl license status`
                                                 # `pdctl license revoke`

.github/workflows/
  publish-podmaker-ce.yml                        # business CI mirror: monorepo → podmaker-sh/podmaker-ce
                                                 # twin of publish-vault-bridge-agent.yml
                                                 # triggers: daily cron, workflow_dispatch,
                                                 # push to main on allowlisted paths, ce-v* tags

scripts/release/
  ce-allowlist.txt                               # paths to copy into CE mirror
  ce-denylist.txt                                # paths to refuse (license-api, Premium/*, T5 YAMLs)
  ce-strip.sh                                    # apply allowlist+denylist; grep -rE
                                                 # "namespace.*Premium|license-api|RequiresPremium"
                                                 # on output tree; fail workflow on leak
  ce-readme.tmpl.md                              # README written into CE mirror after strip
```

## 6. Runtime behaviour

### Boot (CP)
1. `License::load()` reads `storage/app/license.jwt`.
2. Missing → OSS mode. Premium endpoints all return `402 PremiumRequired`.
3. Present → Ed25519 verify against embedded public key.
4. `exp` past → premium disabled, banner.
5. `tier`+`features` cached 1h in Redis.

### Phone-home (cron, 24h)
- POST `https://api.podmaker.sh/v1/license/verify` with `{install_uuid, fingerprint, license_hash}`.
- Response: `{revoked: bool, tier: ..., features: [...], grace_until: ...}`.
- Revoked + grace_until past → premium hard-disabled.
- Unreachable → 72h soft grace, then 7-day grace, then hard disable. Banner counts down.

### Premium method call
- Controller hits `PremiumClient::call('ai-topology-planner.plan', $args)`.
- Client checks license cache (1h TTL).
- If allowed: `POST https://api.podmaker.sh/premium/ai-topology-planner/plan` with `Authorization: Bearer <license.jwt>`.
- License server re-verifies tier + revoked, runs the work, returns result.
- If denied: throw `PremiumRequiredException`; middleware turns it into a 402 with upgrade URL.

## 7. Anti-bypass hardening

- **Cert pin** `*.podmaker.sh` in CP outbound calls (no MITM redirect to fake license server).
- **Embed public key in the binary**, not in a config file (no swap-the-key attack).
- **Phone-home payload signed** by `install_uuid`'s ephemeral key (no replay).
- **Server-side feature ID logging** — every premium call ties to `install_uuid`; abnormal volume triggers panel alert.
- **No client-side feature flag truthy/falsy in source**; method existence depends on `composer.json` profile (premium client classes ship in every build, but they short-circuit to `PremiumRequired` without a valid license).
- **Source obfuscation is not a goal** — security must hold with the source public.
- **CE mirror leak guard** — `ce-strip.sh` runs after the allowlist copy and before commit:
  `grep -rlE 'namespace[^;]*Premium|license-api|RequiresPremium|license\.podmaker\.sh' dst/`
  must return zero hits, otherwise the workflow fails with the offending file list. This is the
  last line of defence against a premium symbol accidentally landing in the public mirror via
  an allowlisted path.
- **PAT scope** — the `PODMAKER_CE_PAT` used by `publish-podmaker-ce.yml` is scoped
  `contents:write` on `podmaker-sh/podmaker-ce` only — no access to other public repos and
  no access back to the private monorepo.

## 8. Tradeoffs the user has accepted

- Self-host CP without `*.podmaker.sh` reachability loses premium features after grace. OSS unaffected — this is the entire point.
- Enterprise air-gap requires a separate SKU and bespoke deal. Pricing must reflect lost revoke-on-demand.

## 9. Open questions for the kickoff session

1. **Pricing tiers** (per-install vs per-seat vs hybrid) — drives `seats` field semantics.
2. **License panel UI** — Filament resource for `License` model with issue/revoke/list?
3. **Migration story** — current OSS users who upgrade — automatic license issue on first paid subscription?
4. **Fingerprint mechanics** — what counts as `machine_id`? (we control the install, can be a CP-generated nonce stored in `storage/app/install.id`).
5. **Multi-region license server** — does `api.podmaker.sh/premium/*` need its own DR plan? (yes, since CPs depend on it).
6. **Free trial** — N-day premium grace for fresh installs without a license? (probably yes for marketing).
7. **CE mirror cadence** — is daily cron + push-on-allowlisted-path enough, or do we want a `ce-v*` tag flow that snapshots into a `releases/` branch on the public repo (parallel to `bridge-v*`)? Affects how `pdctl --self-update` finds the latest CE build.
8. **CE issue forwarding** — file issues on `podmaker-ce` directly or only via a "report bug" link back to the private monorepo? Bridge mirrors say "filed against this mirror, forwarded by maintainers" — same here?

## 10. Out of scope (do not build in the kickoff session)

- Client-side code obfuscation.
- Anti-debugger / anti-RE hardening of the CP binary.
- A second license format for plugins (premium catalogue is the only product).
- Multi-tenant license keys (one install = one license).
