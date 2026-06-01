# Self-Deploy Flow — First-Time PodMaker Production Bring-Up

How to take PodMaker from "nothing exists" to "panel serving traffic"
when *you are the first installation in the world* — no hosted
`app.podmaker.sh` to bootstrap against, no upstream panel to mint
credentials for you. Everything starts from a **local checkout of this
repository on the operator's workstation**.

Read alongside:
- `docs/release/release-flow.md` — how new agent / bridge releases reach a *running* CP.
- `infra/topology/ops.yaml` — the PodMaker-managing-PodMaker topology declaration.
- `apps/orchestrator/internal/selfdeploy/` — the self-update controller.

---

## 1. The chicken-and-egg problem

A normal PodMaker user runs `curl …app.podmaker.sh/install/bootstrap | sh`.
That URL is served by *somebody else's* control plane.

You are bringing up the very first control plane. There is no upstream
panel. The platform has to bootstrap itself from:
- this repo, cloned on your laptop;
- a public OCI image registry (GHCR by default, or your own mirror);
- and exactly one piece of compute (a VPS, a provider account, or both).

Once Tier-0 lands, the panel that comes up **is** the panel — `pdctl`
points at it, self-update takes over, you never SSH the host again
for normal releases.

---

## 2. The three deploy paths

| Tier | Started from | Host provisioner | Stack after install | Pick when |
| --- | --- | --- | --- | --- |
| **Tier-0** Self-host on existing VPS | Local repo + SSH | Operator already provisioned the box | `docker-compose.bootstrap.yml` | VPS exists; one panel, no HA |
| **Tier-1** Provider-bootstrap from laptop | Local repo + provider API token | `pdctl bootstrap` locally calls Hetzner / DO / AWS / … | Same stack as Tier-0 | No VPS yet; one panel, no HA |
| **Tier-2** Self-managed topology | The Tier-0/1 panel | The CP's own orchestrator | Full `infra/topology/ops.yaml` (HA, blue/green) | Production scale-out |

Promotion is always **0/1 → 2** — Tier-2 cannot be the first install
because it requires a running orchestrator + Temporal to drive the
roll-out.

---

## 3. Prereqs (all tiers)

On the **operator workstation**:
- This repo cloned.
- Go ≥ 1.26 (only if you need to rebuild `pdctl` from source; the
  Makefile cross-compiles release binaries).
- `make`, `ssh`, `rsync`, `curl`, `git`.
- A DNS zone you control (Cloudflare or any Caddy-supported DNS-01
  provider).

For the **CP image source**: by default we pull
`ghcr.io/podmaker-sh/control-plane:<tag>` etc. — these are public.
For air-gapped installs see §7.

---

## 4. Tier-0 — Self-host on an existing VPS

You already provisioned a Linux host (Hetzner Cloud console, AWS
Lightsail, a Raspberry Pi, anything). You have `root` or `sudo` over
SSH and a public IP.

### 4.1 Provision DNS first

Point `panel.<your-apex>` (`A` / `AAAA`) at the VPS IP. Caddy obtains
the Let's Encrypt cert via DNS-01 — make sure the DNS API token you
will give Caddy can write under the apex.

### 4.2 Push the bootstrap bundle from your laptop

```sh
# from the repo root on your laptop
make deploy-bootstrap \
    SSH_TARGET=root@1.2.3.4 \
    DOMAIN=panel.acme.io \
    EMAIL=ops@acme.io
```

`make deploy-bootstrap` (Makefile target, see §4.4) does:

1. `rsync` ships these files to `/opt/podmaker/` on the host:
   - `deploy/docker-compose.bootstrap.yml`        → `docker-compose.yml`
   - `deploy/caddy/Caddyfile`                     → `caddy/Caddyfile`
   - `deploy/bootstrap-templates/caddy.env.template`
   - `deploy/bootstrap-templates/control-plane.env.template`
   - `scripts/bootstrap/install-podmaker.sh`
   - `infra/vault/openbao/{config.hcl,entrypoint.sh}`
2. SSH-execs `install-podmaker.sh --domain $DOMAIN --email $EMAIL`
   on the host. Because the script and compose file are already
   present locally on the host, the installer skips the
   `curl …app.podmaker.sh/install/bootstrap-files` fetch step.
3. The installer:
   - installs Docker Engine if missing;
   - generates `APP_KEY`, `DB_PASSWORD`, `OPENBAO_META_TOKEN`,
     `PODMAKER_INTERNAL_TOKEN`;
   - fetches `checksums.txt` from GHCR / Releases for the agent SHAs
     (skippable in air-gapped mode);
   - `docker compose up -d`;
   - runs `php artisan podmaker:bootstrap --domain --email --noninteractive`
     inside the CP container → migrations + ops workspace + admin user
     + first magic-link email.

### 4.3 Manual fallback (no Makefile)

If you can't run `make` against the host (CI runner, restricted
network), the same flow with two commands:

```sh
# from the repo root on your laptop
rsync -avz --delete \
    --include='deploy/***' \
    --include='scripts/bootstrap/***' \
    --include='infra/vault/openbao/***' \
    --exclude='*' \
    ./ root@1.2.3.4:/opt/podmaker-src/

# on the VPS
ssh root@1.2.3.4 'cd /opt/podmaker-src && \
    sudo PODMAKER_SOURCE_BASE=file:///opt/podmaker-src \
         scripts/bootstrap/install-podmaker.sh \
            --domain panel.acme.io --email ops@acme.io'
```

`PODMAKER_SOURCE_BASE=file://…` tells the installer to copy the
compose + Caddyfile + templates from the local rsync'd tree instead
of `curl`-ing them.

### 4.4 What `make deploy-bootstrap` does

The target exists in the root `Makefile` (`make deploy-bootstrap`). It:

- Refuses if `SSH_TARGET` / `DOMAIN` / `EMAIL` are empty.
- `rsync`s the bootstrap bundle.
- SSH-execs the installer with `PODMAKER_SOURCE_BASE=file:///opt/podmaker-src`.
- Streams installer output back to your terminal.
- Exit code mirrors the installer's exit code.

### 4.5 First login

```
Panel:  https://panel.acme.io
Login:  open the magic-link in your email (or the URL printed by the installer)
```

You now have a single-host PodMaker. **The platform exists**. Stop here
if that's enough. Otherwise go to Tier-2 (§6).

### 4.6 Re-running

`install-podmaker.sh` is idempotent — re-running upgrades the stack in
place. Re-run after:
- you tag a new agent release (refreshes SHAs);
- you promote to Tier-2 (stamps the step-ca fingerprint);
- you change any of the template tokens.

The re-run is also how you push hot-patched compose / Caddyfile changes
during the bootstrap window before Tier-2 takes over.

---

## 5. Tier-1 — Provider-bootstrap from your laptop

You don't have a VPS yet. Local `pdctl` (built from this repo) talks
**directly** to a provider's API, provisions the VM, then runs the
same Tier-0 installer through cloud-init. No upstream panel involved.

### 5.1 Build `pdctl` locally

```sh
make build-ctl                       # → bin/podmakerctl
export PATH=$PWD/bin:$PATH           # or copy to /usr/local/bin
pdctl --version
```

### 5.2 Provide provider credentials locally

`pdctl bootstrap` reads provider credentials from environment variables
(scoped to the chosen provider). There is no panel to read OpenBao
from yet — the operator pastes the token into the shell once.

```sh
# Hetzner example
export HCLOUD_TOKEN=xxx                              # provider API
export PODMAKER_DNS_PROVIDER=cloudflare              # for panel.<apex> A record
export CLOUDFLARE_API_TOKEN=yyy
```

Per-provider env var list lives in
`packages/providers/<name>/connection.yaml`.

### 5.3 Bootstrap command

```sh
pdctl bootstrap \
    --provider hetzner \
    --region nbg1 \
    --plan cx21 \
    --domain panel.acme.io \
    --email ops@acme.io \
    --apex acme.io \
    --source ./                       # path to this repo on your laptop
```

What it does:

1. **Provision the VM** through `packages/providers/hetzner` directly
   (no `cloud-broker` in the loop — there is no CP yet to host one).
2. **Create the DNS records** (`panel.acme.io`,
   `agents.acme.io`, `ca.acme.io`, …) through
   `packages/providers/cloudflare`.
3. **Render cloud-init** from `scripts/bootstrap/cloud-init.yml`. The
   `install.sh` body is inlined from the local repo so the VM never
   reaches out to `app.podmaker.sh`.
4. **Wait for SSH** on the new VM.
5. **rsync + install** (same as Tier-0 §4.2) over the SSH connection
   cloud-init set up.
6. **Print the magic-link URL** when the panel is healthy.

### 5.4 Supported providers (Tier-1)

Every adapter under `packages/providers/<name>/` whose
`bootstrap_capable: true` flag is set. Today:

```
hetzner          (Hetzner Cloud)
digitalocean
aws              (EC2)
gcp              (Compute Engine)
azure            (VMs)
linode           (Akamai)
vultr
oci              (Oracle Cloud)
scaleway
contabo
upcloud
kamatera
leaseweb
equinix          (bare metal — no cloud-init, manual install fallback)
byossh           (any host reachable over SSH — same path as Tier-0)
```

`byossh` is the escape hatch — if your provider isn't supported, use
the provider's web console to mint a VM and feed its IP to:

```sh
pdctl bootstrap --provider byossh --ssh-target root@1.2.3.4 \
    --domain panel.acme.io --email ops@acme.io
```

…which is functionally identical to Tier-0 but driven through `pdctl`
so you get the same logging + retry semantics.

### 5.5 Provider credentials after Tier-1

After the panel is up, copy your local env vars into the panel
`Vaults → Connections` admin so subsequent operations (resize, snapshot,
Tier-2 multi-region) can read them. The Tier-1 bootstrap credentials
are **not** persisted anywhere automatically — by design, they leave
your laptop only when the CP is healthy and you can store them in
OpenBao with proper scoping.

---

## 6. Tier-2 — Promote to self-managed topology

Now there's a CP — `panel.<your-apex>` — running Tier-0/1 stack. The
platform manages itself going forward. The declared topology is
`infra/topology/ops.yaml`.

### 6.1 Why Tier-2

- **HA** — every `kind: deployment` node carries `replicas: N` (default
  ≥ 2 for `critical` services).
- **Blue/green self-update** — `apps/orchestrator/internal/selfdeploy`
  flips the inactive colour, waits for `canary_seconds`, then promotes.
  Auto-revert if `canary_error_rate_max` is breached (see
  `apps/control-plane/config/self_deploy.php`).
- **Multi-region** — leaf NATS gateways and per-region agent gateways
  drop in by declaring `region:` on the topology node.
- **Backups + DR** — `kind: managed` nodes (Postgres, OpenBao) carry
  `backup: { schedule: ..., retention_days: ... }` honoured by the
  `migration-engine` job.

### 6.2 Promotion

```sh
# Point pdctl at YOUR panel (the one you just bootstrapped).
pdctl auth login --panel https://panel.acme.io

# Import the platform's own topology into the ops workspace.
pdctl topology import-ops --workspace ops --file infra/topology/ops.yaml

# Diff what would change vs the running bootstrap stack.
pdctl plan diff --workspace ops

# Roll it out. Default strategy is blue/green per the topology node.
pdctl plan apply --workspace ops --canary
```

The orchestrator walks nodes in dependency order, runs the
`migration-engine` job once per release, then swaps each deployment
node's inactive colour. The CP itself swaps **last** — by the time
the orchestrator restarts it, the new colour is already serving.

### 6.3 The `ops.yaml` shape (summary)

```yaml
apiVersion: podmaker.sh/v1
kind: Topology
metadata:
  name: podmaker-ops
  workspace: ops
spec:
  - node_key: caddy              # edge / TLS
  - node_key: website            # marketing apex
  - node_key: control-plane      # Laravel + Filament (replicas: 3)
  - node_key: orchestrator       # Temporal worker
  - node_key: agent-gateway      # WSS to enrolled agents
  - node_key: cloud-broker       # provider abstraction (Tier-1 now panel-driven)
  - node_key: vault-broker       # OpenBao tenant cluster manager
  - node_key: topology-planner   # tenant topology compiler
  - node_key: build-service
  - node_key: repo-scanner
  - node_key: db-controller
  - node_key: lb-controller
  - node_key: mesh-controller
  - node_key: cache-controller
  - node_key: metrics-consumer
  - node_key: event-bridge
  - node_key: migration-engine   # kind: job, run_once_per_release: true
  - node_key: postgres           # kind: managed, persistent_volume
  - node_key: redis              # kind: managed
  - node_key: openbao-meta       # kind: managed, nightly backup
  - node_key: openbao-tenant     # kind: managed
  - node_key: temporal           # kind: managed
  - node_key: nats               # plus per-region leaves
  - node_key: step-ca            # kind: managed, root volume retained
  - node_key: zot                # OCI registry
```

`kind: deployment` → blue/green rolling pair.
`kind: managed` → operator-supervised state; orchestrator only
health-checks + emits backup jobs.
`kind: job` → run-to-completion per release (DB migrations).

### 6.4 Re-run the host installer once after Tier-2

Tier-2 brings up `step-ca`. The bootstrap-stage `control-plane.env`
has `PODMAKER_STEP_CA_FINGERPRINT=` empty (it didn't exist yet). Run
the host installer **one more time** so it extracts the fingerprint
from the now-running container and upserts the env file:

```sh
ssh root@1.2.3.4 'cd /opt/podmaker-src && \
    sudo PODMAKER_SOURCE_BASE=file:///opt/podmaker-src \
         scripts/bootstrap/install-podmaker.sh \
            --domain panel.acme.io --email ops@acme.io'
```

The installer detects step-ca via `docker compose ps`, runs
`step certificate fingerprint`, upserts the env, restarts CP. After
this run, every freshly minted agent install script has real cert
pinning — no `PODMAKER_AGENT_ALLOW_UNVERIFIED=1` bypass.

### 6.5 Execution backends

`apps/orchestrator/internal/selfdeploy/k8s_executor.go` ships the
Kubernetes executor. Docker (compose) is the default for single-host
Tier-2 ("HA on one big VM"). Select with `--executor`:

```sh
pdctl plan apply --workspace ops --executor docker   # default
pdctl plan apply --workspace ops --executor k8s --cluster $(pdctl cluster list --json | jq -r '.[0].id')
```

`pdctl cluster list` enumerates managed clusters registered through
`Clusters → Connections` in the panel.

### 6.6 Self-update from Tier-2 onwards

```sh
# Tag the platform release (composite of every service image).
make release-platform TAG_VERSION=v1.2.0     # see docs/release/release-flow.md

# Workflow updates vault.bridge.channels.stable.version.
# The orchestrator's reconcile loop notices and:
#   1. Renders ops.yaml with {{ release }}=v1.2.0.
#   2. Builds the diff vs the running deployment.
#   3. Runs blue/green per node with canary auto-revert.
```

No operator action required for normal rollouts. Canary thresholds:

```php
// apps/control-plane/config/self_deploy.php
'canary_seconds'        => 300,
'canary_error_rate_max' => 0.02,    // 2% 5xx triggers auto-revert
```

---

## 7. Air-gapped / private-registry installs

By default the bootstrap pulls images from `ghcr.io/podmaker-sh/*` and
agent tarballs from `github.com/podmaker-sh/releases`. To run without
public-internet egress:

1. **Mirror the images** to your registry:
   ```sh
   for svc in control-plane website orchestrator agent-gateway cloud-broker \
              vault-broker topology-planner build-service repo-scanner \
              db-controller lb-controller mesh-controller cache-controller \
              metrics-consumer event-bridge migration-engine; do
       skopeo copy \
           docker://ghcr.io/podmaker-sh/$svc:latest \
           docker://registry.internal/podmaker/$svc:latest
   done
   ```
2. **Mirror the agent + bridge tarballs** into your release server
   (any HTTPS host that serves the files at the same relative paths).
3. **Point the installer at the mirrors**:
   ```sh
   make deploy-bootstrap \
       SSH_TARGET=root@10.0.0.5 \
       DOMAIN=panel.internal \
       EMAIL=ops@internal \
       IMAGE_REGISTRY=registry.internal/podmaker \
       PODMAKER_AGENT_RELEASE_BASE=https://releases.internal/podmaker/agent
   ```
4. **Override checksums in `control-plane.env`** so `AgentReleaseManifest`
   doesn't try to reach `releases.internal` for every install:
   ```
   PODMAKER_AGENT_SHA256_AMD64=…
   PODMAKER_AGENT_SHA256_ARM64=…
   ```
   (`make agent-sha-env TAG=agent-vX.Y.Z` emits the snippet.)

The installer respects `PODMAKER_SOURCE_BASE=file://…` for the compose
+ Caddyfile + templates (it never `curl`s `app.podmaker.sh` once that
env var is set).

---

## 8. Backups + disaster recovery (all tiers)

Backups are taken from `kind: managed` state nodes. Disk defaults to
`SELF_DEPLOY_BACKUP_DISK` (default `local`):

```
node            schedule        retention
─────────────── ─────────────── ─────────
postgres        0 2 * * *       30 days
openbao-meta    0 3 * * *       30 days
openbao-tenant  0 3 * * *       30 days
step-ca volume  weekly          90 days
```

Switch the disk to S3 / Backblaze / Wasabi by setting
`SELF_DEPLOY_BACKUP_DISK` to a configured Laravel filesystem.

### 8.1 Restore

```sh
pdctl restore --workspace ops \
    --target postgres \
    --snapshot 2026-05-31T02:00:00Z
```

The orchestrator suspends `migration-engine`, runs the restore as a
`job`, replays Temporal history past the restore point, then resumes.

### 8.2 Cold rebuild from backup

Host obliterated:

1. Provision a fresh VPS (Tier-0 §4) **or** run `pdctl bootstrap`
   (Tier-1 §5).
2. Before the first `docker compose up`, drop your archived
   `secrets/control-plane.env` + `data/openbao-meta/` into
   `/opt/podmaker/`.
3. The installer detects existing secrets and reuses them.
4. Promote back to Tier-2 with §6.2 + restore Postgres from §8.1.

---

## 9. Putting it together — the canonical first deploy

```sh
# 0. Local prereqs
git clone https://github.com/podmaker-sh/hostingmanager.git
cd hostingmanager
make build-ctl

# 1a. You already have a VPS — Tier-0
make deploy-bootstrap \
    SSH_TARGET=root@1.2.3.4 \
    DOMAIN=panel.acme.io \
    EMAIL=ops@acme.io

# 1b. You don't have a VPS — Tier-1 (alternative to 1a)
export HCLOUD_TOKEN=xxx
export CLOUDFLARE_API_TOKEN=yyy
pdctl bootstrap --provider hetzner --region nbg1 --plan cx21 \
    --domain panel.acme.io --email ops@acme.io --apex acme.io --source ./

# 2. Visit https://panel.acme.io, open the magic-link from the email.

# 3. Configure Cloudflare + step-ca connections in the admin.

# 4. Promote to Tier-2.
pdctl auth login --panel https://panel.acme.io
pdctl topology import-ops --workspace ops --file infra/topology/ops.yaml
pdctl plan diff   --workspace ops
pdctl plan apply  --workspace ops --canary

# 5. Re-run the host installer to stamp the now-existing step-ca
#    fingerprint into control-plane.env (Tier-2 only; skip for Tier-0).
ssh root@1.2.3.4 'cd /opt/podmaker-src && \
    sudo PODMAKER_SOURCE_BASE=file:///opt/podmaker-src \
         scripts/bootstrap/install-podmaker.sh \
            --domain panel.acme.io --email ops@acme.io'

# 6. Cut your first agent release through the normal flow.
make release-agent
```

You now have a self-managing PodMaker. Subsequent releases ride the
self-update controller — no host SSH.

---

## 10. Quick reference

```
# Local prereqs
git clone <repo> && cd hostingmanager && make build-ctl

# Tier-0  (operator-provisioned VPS, install from local repo)
make deploy-bootstrap SSH_TARGET=root@HOST DOMAIN=DOM EMAIL=MAIL
# manual fallback:
rsync deploy/ scripts/bootstrap/ infra/vault/openbao/ root@HOST:/opt/podmaker-src/
ssh root@HOST 'sudo PODMAKER_SOURCE_BASE=file:///opt/podmaker-src \
    /opt/podmaker-src/scripts/bootstrap/install-podmaker.sh \
    --domain DOM --email MAIL'

# Tier-1  (pdctl provisions VPS from your laptop)
pdctl bootstrap --provider hetzner       --region nbg1     --plan cx21       --domain DOM --email MAIL --apex APEX --source ./
pdctl bootstrap --provider aws           --region eu-central-1 --plan t4g.medium --domain DOM --email MAIL --apex APEX --source ./
pdctl bootstrap --provider byossh        --ssh-target root@1.2.3.4              --domain DOM --email MAIL --apex APEX --source ./

# Tier-2  (only after panel is up)
pdctl auth login --panel https://panel.DOM
pdctl topology import-ops --workspace ops --file infra/topology/ops.yaml
pdctl plan diff   --workspace ops
pdctl plan apply  --workspace ops --canary
pdctl plan apply  --workspace ops --executor k8s --cluster CLUSTER_ID

# Stamp step-ca fingerprint after Tier-2 (re-run installer)
ssh root@HOST 'sudo PODMAKER_SOURCE_BASE=file:///opt/podmaker-src \
    /opt/podmaker-src/scripts/bootstrap/install-podmaker.sh \
    --domain DOM --email MAIL'

# Restore
pdctl restore --workspace ops --target postgres --snapshot ISO_TS
```

---

## 11. Open work — what doesn't exist yet

Honest accounting so the doc tracks reality, not aspiration:

- `make deploy-bootstrap` — **not yet in the Makefile.** The plain
  `rsync + ssh` fallback (§4.3) works today. Adding the target is a
  ~30-line shell wrapper; tracked alongside this doc.
- `pdctl bootstrap` — **not yet implemented.** `cmd/podmakerctl/up.go`
  is the closest sibling; Tier-1 today means using the provider's web
  console + Tier-0 against the resulting IP. The CLI command will land
  with the first Tier-1 provider (likely Hetzner) wired in.
- `PODMAKER_SOURCE_BASE=file://…` handling — currently
  `install-podmaker.sh` accepts the env var name; the file-scheme
  branch is one `case` in `fetch()`. Tracked.
- `bootstrap_capable: true` flag in `connection.yaml` — convention
  proposed in §5.4 but not yet asserted in code.

Use this list as the punch list for "make the doc literally true".

---

## 12. Related docs

- `docs/release/release-flow.md` — release tagging + SHA propagation.
- `docs/release/podmaker-sh-repo-bootstrap.md` — provisioning the public
  release repo + `RELEASES_PAT`.
- `docs/release/homebrew-tap-bootstrap.md` — `brew install podmaker`.
- `infra/topology/ops.yaml` — the canonical platform self-deploy spec.
- `apps/orchestrator/internal/selfdeploy/selfdeploy.go` — controller
  walking the topology nodes.
- `apps/control-plane/config/self_deploy.php` — canary thresholds.
