# Release Flow — `podmaker-agent` & `vault-bridge-agent`

End-to-end developer guide for cutting a new agent or bridge release,
how the artifacts reach a production control plane, and how the SHA-256
checksums + step-ca fingerprint get into the running stack **without
touching `control-plane.env` after the first install**.

---

## 1. Components

| Piece | Where | Purpose |
|------|-------|---------|
| `make release-agent` / `release-bridge` | `Makefile` + `scripts/release/next-tag.sh` | Compute next tag, create + push, fire workflow |
| `.github/workflows/release-agent.yml` | GitHub Actions | Cross-compile `linux/amd64,arm64`, sign with cosign, upload to `podmaker-sh/releases` |
| `.github/workflows/release-vault-bridge.yml` | GitHub Actions | Same shape for bridge (also `darwin`, `windows`, `.deb`, `.rpm`) |
| `podmaker-sh/releases` GitHub release | Public assets | `*.tar.gz`, `*.tar.gz.sha256`, `*.tar.gz.sig`, `*.tar.gz.pem` per arch |
| `App\Services\Agent\AgentReleaseManifest` | `apps/control-plane/app/Services/Agent/` | Fetches per-arch `.sha256` at runtime, 5-min cache |
| `InstallController::agent` | `apps/control-plane/app/Http/Controllers/InstallController.php` | Stamps the install script with the resolved SHA + step-ca fingerprint |
| `scripts/bootstrap/install-podmaker.sh` | Host bootstrap | Pulls SHAs into `control-plane.env`, extracts step-ca fingerprint after stack is up |

---

## 2. Cutting a release

### 2.1 Auto-bump (default: patch)

```sh
make release-agent             # last agent-vX.Y.Z + patch
make release-bridge BUMP=minor # last bridge-vX.Y.Z + minor
make release-agent BUMP=major  # last agent-vX.Y.Z + major
```

The Makefile target:
1. Resolves the next tag via `scripts/release/next-tag.sh <prefix> <bump>`.
2. Refuses if the working tree is dirty (override with `DIRTY_OK=1`).
3. Refuses if the resolved tag already exists.
4. `git tag -a <tag> -m "release <tag>"`.
5. `git push origin refs/tags/<tag>` — this fires the matching workflow.

### 2.2 Explicit version

```sh
make release-agent TAG_VERSION=v0.3.0
make release-bridge TAG_VERSION=v1.0.0-rc1
```

Both `0.3.0` and `v0.3.0` are accepted; the script normalises.

### 2.3 Dry run

```sh
make release-agent  DRY_RUN=1 DIRTY_OK=1   # print intended tag, do not push
make release-tag-dry KIND=agent BUMP=minor # print resolved tag only
make release-tag-dry KIND=bridge
```

### 2.4 Override remote

```sh
make release-agent REMOTE=upstream
```

Default: `origin`.

### 2.5 Workflow dispatch (manual, no tag)

Both workflows also accept `workflow_dispatch` with a `version` input.
Use this only when you need to re-run the build against an existing tag
or branch — normal releases should go through `make release-*`.

```sh
gh workflow run release-agent.yml -f version=v0.3.0
gh workflow run release-vault-bridge.yml -f version=v1.0.0
```

---

## 3. What the workflows produce

`podmaker-sh/releases` GitHub Release, tag = `agent-vX.Y.Z` (or `bridge-vX.Y.Z`):

```
podmaker-agent-linux-amd64.tar.gz
podmaker-agent-linux-amd64.tar.gz.sha256
podmaker-agent-linux-amd64.tar.gz.sig    # cosign keyless OIDC
podmaker-agent-linux-amd64.tar.gz.pem
podmaker-agent-linux-arm64.tar.gz
podmaker-agent-linux-arm64.tar.gz.sha256
podmaker-agent-linux-arm64.tar.gz.sig
podmaker-agent-linux-arm64.tar.gz.pem
```

The `.sha256` files contain a bare hex digest (one line). The
`AgentReleaseManifest` service depends on this format — keep the
workflow output consistent with the Makefile's `agent-release` target.

Vault-bridge releases add `darwin-{amd64,arm64}`, `windows-amd64.exe`,
plus `.deb` / `.rpm` packages built via `nfpm` and a multi-arch
container image pushed to GHCR.

---

## 4. How a running control plane picks up a new release

No `control-plane.env` edit needed after the first install.

```
operator pushes tag agent-vX.Y.Z
   │
   ├── release-agent.yml builds binaries
   │    └─> uploads to podmaker-sh/releases/tag/agent-vX.Y.Z
   │        + updates the moving 'latest' release
   │
   └── on the next install-script render OR within 5 min for cached lookups:
        InstallController::agent calls
          AgentReleaseManifest::sha256(arch)
            ├─ env override?  (PODMAKER_AGENT_SHA256_AMD64 / _ARM64) — wins if set
            └─ HTTP GET <release_base>/podmaker-agent-linux-<arch>.tar.gz.sha256
                 -> Cache::remember(podmaker.agent.sha256.<arch>, 300s)
                 -> returns lowercase 64-char hex
        => baked into the per-host install script
```

`release_base` resolves from `PODMAKER_AGENT_RELEASE_BASE`
(default `https://github.com/podmaker-sh/releases/releases/latest/download`).

The cache TTL is 300 seconds. To roll out faster on a freshly tagged
release without waiting:

```sh
docker compose -f docker-compose.prod.yml exec control-plane \
    php artisan cache:forget 'podmaker.agent.sha256.amd64'
docker compose -f docker-compose.prod.yml exec control-plane \
    php artisan cache:forget 'podmaker.agent.sha256.arm64'
```

### Override path (air-gapped installs)

When the control plane has no outbound internet, set
`PODMAKER_AGENT_SHA256_AMD64` / `_ARM64` in `control-plane.env`. The
service short-circuits and returns the env value without making any
HTTP call.

`make agent-sha-env TAG=agent-vX.Y.Z` prints a paste-ready snippet:

```sh
make agent-sha-env TAG=agent-v0.3.0
# PODMAKER_AGENT_SHA256_AMD64=ab12…
# PODMAKER_AGENT_SHA256_ARM64=cd34…
```

---

## 5. step-ca root fingerprint

Different lifecycle from the SHA flow — fingerprint is generated on the
**operator's** host the first time step-ca boots, not at release time.

### 5.1 Bootstrap stack (`docker-compose.bootstrap.yml`)

No step-ca. The installer writes the fingerprint key empty, and the
`InstallController` sets `PODMAKER_AGENT_ALLOW_UNVERIFIED=1` on every
agent install script (dev / pre-prod only).

### 5.2 Full prod stack (`docker-compose.prod.yml`)

step-ca is included. After the operator brings the prod stack up, they
re-run `install-podmaker.sh`:

```sh
sudo /opt/podmaker/install-podmaker.sh --domain panel.acme.io --email ops@acme.io
```

The script (idempotent) does:
1. `docker compose ps --services | grep -qx step-ca` — present in prod stack.
2. `docker compose exec step-ca step certificate fingerprint /home/step/certs/root_ca.crt`.
3. `upsert_env_var control-plane.env PODMAKER_STEP_CA_FINGERPRINT <hex>`.
4. `docker compose restart control-plane`.

`InstallController::agent` then stamps the real fingerprint into every
subsequent install script — no `ALLOW_UNVERIFIED` bypass.

---

## 6. Per-host agent install

Operator visits `Servers → Adopt` in the panel and runs the rendered
one-liner on the target VM:

```sh
curl -fsSL https://panel.acme.io/install/agent?arch=amd64 | sudo sh
```

The script that comes back has these lines stamped at the top by
`InstallController::agent`:

```sh
export PODMAKER_AGENT_ID="…"
export PODMAKER_AGENT_URL="https://github.com/podmaker-sh/releases/releases/latest/download/podmaker-agent-linux-amd64.tar.gz"
export PODMAKER_AGENT_SHA256="<resolved by AgentReleaseManifest>"
export PODMAKER_GATEWAY_URL="wss://agents.acme.io"
export PODMAKER_STEP_CA_URL="https://ca.acme.io"
export PODMAKER_STEP_CA_FINGERPRINT="<upserted by installer post-stack-up>"
export PODMAKER_AGENT_ALLOW_UNVERIFIED="0"   # 1 only if SHA or fingerprint missing
```

The rest of `scripts/bootstrap/install.sh` then:
1. Downloads the tarball + verifies SHA-256.
2. Pins step-ca's root cert via the fingerprint.
3. Calls `/1.0/sign` with the JWT one-time token to obtain a leaf cert.
4. Writes `/etc/podmaker-agent/agent.env` + starts `podmaker-agent.service`.

---

## 7. Local cross-compile (no GitHub Actions)

Useful for testing tarball shape, signing, or air-gapped publishing.

```sh
make agent-release  VERSION=v0.3.0   # → bin/release/podmaker-agent-linux-{amd64,arm64}.tar.gz(.sha256)
make bridge-release VERSION=v1.0.0   # → bin/release/podmaker-vault-bridge-<plat>(.tar.gz/.sha256)
make bridge-packages VERSION=v1.0.0  # → .deb + .rpm via nfpm
make bridge-image    VERSION=v1.0.0  # → docker image, local
```

`VERSION` is also embedded into the binary via `-ldflags -X main.version`.
`make build-ctl-release VERSION=v1.2.3` does the same for `podmakerctl`.

---

## 8. Rolling back

GitHub Releases are append-only from our side. To roll back:

1. **Move `latest`** — re-publish the previous tag as `latest` either by
   deleting the new release on `podmaker-sh/releases` (manual via web
   UI) or by re-running the workflow against the older tag with
   `workflow_dispatch`.
2. **Bust the control-plane cache** — `cache:forget` both
   `podmaker.agent.sha256.amd64` and `.arm64` so the next install
   script picks up the rolled-back SHA within 5 minutes.
3. **Pin via env** — if you need a precise version that is not `latest`,
   set `PODMAKER_AGENT_RELEASE_BASE` to the specific tag's asset URL and
   restart the CP:
   ```
   PODMAKER_AGENT_RELEASE_BASE=https://github.com/podmaker-sh/releases/releases/download/agent-v0.2.0
   ```

In-flight agents that already enrolled keep running on whichever binary
they downloaded; the rollback only affects *new* adoptions.

---

## 9. Quick reference

```
# release
make release-agent                          # patch
make release-agent BUMP=minor               # minor
make release-bridge TAG_VERSION=v1.0.0      # explicit
make release-agent DRY_RUN=1 DIRTY_OK=1     # preview

# resolve
make release-tag-dry KIND=agent             # print next tag
make release-tag-dry KIND=bridge BUMP=major

# local build
make agent-release  VERSION=v0.3.0
make bridge-release VERSION=v1.0.0

# panel: bust cache after a release
php artisan cache:forget podmaker.agent.sha256.amd64
php artisan cache:forget podmaker.agent.sha256.arm64

# operator: stamp step-ca fingerprint into control-plane.env
sudo /opt/podmaker/install-podmaker.sh --domain $DOMAIN --email $EMAIL
```

## 10. Related docs

- `docs/release/homebrew-tap-bootstrap.md` — operator-side `brew install podmaker`.
- `docs/release/podmaker-sh-repo-bootstrap.md` — provisioning the `podmaker-sh/releases` repo + `RELEASES_PAT`.
- `docs/release/vault-bridge-release-secrets.md` — GPG signing keys for `.deb` / `.rpm` packages.
