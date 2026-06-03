# PodMaker CE Mirror — Implementation Plan

> **Status:** draft (2026-06-02). Not started.
> **Owner:** TBD (release engineering).
> **Parent:** [`self-hosted-licensing-plan.md`](./self-hosted-licensing-plan.md) §2 + §2a.
> **Sibling plans:** [`license-api-plan.md`](./license-api-plan.md), [`panel-licensing-ui-plan.md`](./panel-licensing-ui-plan.md), [`podmaker-ce-install-plan.md`](./podmaker-ce-install-plan.md), [`podmaker-ce-pairing-plan.md`](./podmaker-ce-pairing-plan.md).

The licensing plan promises a public Community Edition (`podmaker-sh/podmaker-ce`)
generated from the private monorepo via the same business-CI mirror pattern already
proven for `apps/vault-bridge-agent` and `apps/podmaker-bridge`. This doc captures the
phase-1 work: the workflow, the strip pipeline, and the bootstrap on the public side.
The mirror does not depend on `license-api` or panel work — once shipped, the OSS distro
is a standalone artefact.

## 0. Kickoff prompt (new session)

> Read `docs/release/podmaker-ce-mirror-plan.md` and implement §3–§6 in order:
> author `scripts/release/ce-allowlist.txt`, `scripts/release/ce-denylist.txt`,
> `scripts/release/ce-strip.sh`, `scripts/release/ce-readme.tmpl.md`, then
> `.github/workflows/publish-podmaker-ce.yml` mirroring the shape of
> `.github/workflows/publish-vault-bridge-agent.yml`. Do not create the public
> `podmaker-sh/podmaker-ce` repo or generate the PAT in this PR — those are §7
> bootstrap steps the human runs. Stop after the workflow is mergeable with the
> leak guard exercised against a `--dry-run` flag. Open questions in §9 are
> blocking.

## 0a. Scope note (locked 2026-06-02)

The CE distribution is **deliberately narrower than "OSS minus
premium"**. Parent doc §0a locks the product shape: single-host
hosting panel + operator-attached VPSes, no multi-cloud, no
bridges. That ripples through this plan as additional denylist
entries (`apps/cloud-broker`, `apps/vault-bridge-agent`,
`apps/podmaker-bridge`, `apps/mesh-controller`, `tiers/t2-managed-*`,
`tiers/t3-k8s-*`, `tiers/t3-multi-host-*`). The allowlist is the
declared OSS surface; the denylist enforces the CE scope on top of
that. See the current files for the authoritative list:
[`scripts/release/ce-allowlist.txt`](../../scripts/release/ce-allowlist.txt) +
[`scripts/release/ce-denylist.txt`](../../scripts/release/ce-denylist.txt).

## 1. Goals + non-goals

| Goal | Why |
|---|---|
| Public OSS mirror of CP, mesh, orchestrator (OSS bits), pdctl (OSS subcmds), shared-go, T1–T3 tier YAMLs | Self-host operators can build without premium symbols on disk |
| Drift-safe: daily cron + push-on-allowlisted-path + `ce-v*` tag | Mirror cannot lag the private repo silently |
| Leak-proof: workflow fails closed if any premium symbol lands in `dst/` | Source of truth for license enforcement stays private |
| One-line install + build instructions on the CE README | Adoption funnel from GitHub stars |

| Non-goal | Reason |
|---|---|
| Closed-source Premium distribution channel | Lives in `license-api-plan.md` (binary release, not source mirror) |
| Air-gap Enterprise bundle generator | Separate SKU, separate runbook |
| CE-specific issue tracker tooling | File issues against the mirror; forwarded by maintainers (§9.Q8 of parent doc) |

## 2. Architecture

```
private monorepo (this repo)
   │
   │ trigger: cron 03:15 UTC, workflow_dispatch, push:main on allowlisted paths, ce-v* tag
   ▼
publish-podmaker-ce.yml
   │
   ├─ checkout src (private monorepo, full clone)
   ├─ checkout dst (public podmaker-sh/podmaker-ce, PAT-auth)
   ├─ scripts/release/ce-strip.sh src dst
   │     ├─ wipe dst tree except .git
   │     ├─ for each line in ce-allowlist.txt: cp -R src/<path> dst/<path>
   │     ├─ for each line in ce-denylist.txt: rm -rf dst/<glob>
   │     ├─ grep -rlE "<premium symbol regex>" dst/ → fail on hit
   │     └─ render ce-readme.tmpl.md → dst/README.md
   ├─ git add -A; if no diff, exit 0
   ├─ git commit -m "Sync podmaker-ce from monorepo @ <sha> [tag <ref>]"
   └─ git push
```

## 3. `scripts/release/ce-allowlist.txt`

Paths copied into the mirror (whitespace-separated path globs, one per line, `#` comments).

```
# control plane (OSS profile)
apps/control-plane/app
apps/control-plane/bootstrap
apps/control-plane/config
apps/control-plane/database
apps/control-plane/public
apps/control-plane/resources
apps/control-plane/routes
apps/control-plane/storage/.gitkeep
apps/control-plane/tests
apps/control-plane/artisan
apps/control-plane/composer.json
apps/control-plane/composer.lock
apps/control-plane/package.json
apps/control-plane/package-lock.json
apps/control-plane/phpunit.xml
apps/control-plane/vite.config.js

# operator-side daemons
apps/vault-bridge-agent
apps/podmaker-bridge

# control plane peers
apps/mesh-controller
apps/orchestrator

# shared Go + protobuf
packages/shared-go
packages/proto

# pdctl (OSS subcommands)
cmd/podmakerctl

# OSS tier YAMLs (T1–T3 only)
tiers/_schema.json
tiers/t1-*.yaml
tiers/t2-*.yaml
tiers/t3-*.yaml

# root-level scaffolding
go.mod
go.sum
LICENSE
.editorconfig
.gitignore

# OSS-safe docs
docs/installer
docs/operator
docs/tier-spec.md
```

## 4. `scripts/release/ce-denylist.txt`

Removed *after* the allowlist copy (so allowlisted dirs can leave premium files behind).

```
# license-api never ships
apps/license-api

# CP premium services
apps/control-plane/app/Services/Premium
apps/control-plane/app/Http/Middleware/RequiresPremium.php
apps/control-plane/app/Console/Commands/LicenseStatusCommand.php
apps/control-plane/config/premium.php

# pdctl license subcommand
cmd/podmakerctl/license.go
cmd/podmakerctl/license_test.go

# enterprise tier YAMLs
tiers/t5-enterprise-*.yaml

# planning + internal docs
docs/release/self-hosted-licensing-plan.md
docs/release/license-api-plan.md
docs/release/panel-licensing-ui-plan.md
docs/release/podmaker-ce-mirror-plan.md
docs/internal

# CI workflows that target private infra
.github/workflows/publish-podmaker-ce.yml
.github/workflows/manifest-trust-check.yml
```

## 5. `scripts/release/ce-strip.sh`

```bash
#!/usr/bin/env bash
# ce-strip.sh — produce a CE mirror tree under $DST from $SRC.
# Usage: ce-strip.sh <SRC> <DST> [--dry-run]
# Exit codes: 0 success / no changes, 2 leak detected, 3 allowlist miss.

set -euo pipefail

SRC=${1:?src path required}
DST=${2:?dst path required}
DRY=${3:-}

ROOT=$(cd "$(dirname "$0")/../.." && pwd)
ALLOW=$ROOT/scripts/release/ce-allowlist.txt
DENY=$ROOT/scripts/release/ce-denylist.txt
TPL=$ROOT/scripts/release/ce-readme.tmpl.md

# 1. wipe dst except .git
find "$DST" -mindepth 1 -not -path "$DST/.git*" -delete

# 2. copy allowlisted paths
while IFS= read -r line; do
    [[ -z "$line" || "$line" == \#* ]] && continue
    for match in "$SRC"/$line; do
        [[ -e "$match" ]] || continue
        rel=${match#$SRC/}
        mkdir -p "$DST/$(dirname "$rel")"
        cp -R "$match" "$DST/$(dirname "$rel")/"
    done
done < "$ALLOW"

# 3. apply denylist
while IFS= read -r line; do
    [[ -z "$line" || "$line" == \#* ]] && continue
    rm -rf "$DST"/$line
done < "$DENY"

# 4. leak guard
LEAK=$(grep -rlE 'namespace[^;]*Premium|license-api|RequiresPremium|license\.podmaker\.sh' "$DST" || true)
if [[ -n "$LEAK" ]]; then
    echo "ce-strip: leak detected — premium symbols in mirror tree:" >&2
    echo "$LEAK" >&2
    exit 2
fi

# 5. render README
SHA=${GITHUB_SHA:-$(git -C "$SRC" rev-parse HEAD)}
sed "s/__MONOREPO_SHA__/$SHA/g" "$TPL" > "$DST/README.md"

[[ "$DRY" == "--dry-run" ]] && exit 0
```

## 6. `scripts/release/ce-readme.tmpl.md`

```markdown
# podmaker-ce — PodMaker Community Edition

Self-hosted, MIT-licensed PodMaker control plane + mesh + orchestrator.
This repo is a **read-only mirror** of the OSS surface of the private
monorepo `podmaker-sh/podmaker`. Refreshed on every push to monorepo
`main`, every `ce-v*` release tag, and once daily at 03:15 UTC.

Last sync: monorepo `__MONOREPO_SHA__`.

## What's here

- Control plane (Laravel) — full panel, no premium symbols
- Mesh controller, orchestrator (OSS bits), vault-bridge-agent, podmaker-bridge
- `pdctl` (OSS subcommands)
- Tier YAMLs for T1–T3

## What's *not* here

Premium server-side features (AI topology planner, anomaly detector,
multi-region scheduler, SAML SSO, audit export, fleet telemetry) bind
to `api.podmaker.sh` and are not shipped as source. The CE binary still
contains the premium *client* shells — they return `402 PremiumRequired`
without a valid license JWT. See <https://podmaker.sh/pricing> for the
Premium/Enterprise SKU details.

## Install

```sh
git clone https://github.com/podmaker-sh/podmaker-ce
cd podmaker-ce/apps/control-plane
composer install
php artisan migrate
php artisan serve
```

Full installer docs in `docs/installer/`.

## Issues + PRs

File against this mirror — forwarded to the monorepo by maintainers.
```

## 7. Workflow shape — `.github/workflows/publish-podmaker-ce.yml`

```yaml
name: publish-podmaker-ce

on:
  schedule:
    - cron: '15 3 * * *'
  workflow_dispatch:
  push:
    branches: [main]
    paths:
      - 'apps/control-plane/**'
      - 'apps/mesh-controller/**'
      - 'apps/orchestrator/**'
      - 'apps/vault-bridge-agent/**'
      - 'apps/podmaker-bridge/**'
      - 'packages/shared-go/**'
      - 'packages/proto/**'
      - 'cmd/podmakerctl/**'
      - 'tiers/t[1-3]-*.yaml'
      - 'tiers/_schema.json'
      - 'scripts/release/ce-*'
      - '.github/workflows/publish-podmaker-ce.yml'
    tags:
      - 'ce-v*'

jobs:
  publish:
    name: sync to podmaker-sh/podmaker-ce
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@v4
        with:
          path: src
      - uses: actions/checkout@v4
        with:
          repository: podmaker-sh/podmaker-ce
          token: ${{ secrets.PODMAKER_CE_PAT }}
          path: dst
      - name: Strip + mirror
        run: scripts/release/ce-strip.sh src dst
        working-directory: ${{ github.workspace }}
      - name: Commit + push
        working-directory: dst
        run: |
          git config user.name  "podmaker-bot"
          git config user.email "bot@podmaker.sh"
          git add -A
          if git diff --cached --quiet; then
              echo "no changes — skipping commit"
              exit 0
          fi
          MSG="Sync podmaker-ce from monorepo @ ${{ github.sha }}"
          if [ "${GITHUB_REF_TYPE:-}" = "tag" ]; then
              MSG="$MSG (tag ${GITHUB_REF_NAME})"
          fi
          git commit -m "$MSG"
          git push
```

## 8. Bootstrap (manual, one-shot)

1. Create empty public repo `podmaker-sh/podmaker-ce` with MIT licence + empty README.
2. Apply branch protection: require linear history, block force-pushes, enforce admins.
3. Generate fine-grained PAT scoped `contents:write` on `podmaker-sh/podmaker-ce` only.
   Store as `PODMAKER_CE_PAT` repo secret on the private monorepo.
4. Dry-run the workflow locally:
   ```sh
   scripts/release/ce-strip.sh . /tmp/ce-test --dry-run
   ```
   Verify the tree under `/tmp/ce-test` builds, runs `php artisan test`, and contains
   no `Premium`/`license-api`/`RequiresPremium` symbols.
5. Run `gh workflow run publish-podmaker-ce.yml` from `main`. Inspect the diff on
   `podmaker-ce` before declaring success.
6. Tag the first snapshot: `git tag ce-v0.1.0 && git push --tags`.

## 9. Open questions

1. Does `pdctl` ship via the CE repo source build *only*, or also via the Homebrew tap
   formula (`brew install podmaker-ce`)? Affects whether we need a parallel
   `release-podmaker-ce.yml` for binaries.
2. Should the OSS profile of CP run without any `Premium*` client class on disk, or
   keep them as shells that return 402? The licensing plan (§7) prefers shells —
   confirm by reading `apps/control-plane/app/Services/Premium/PremiumClient.php`
   once it exists.
3. Do we mirror `docs/operator/` wholesale or curate? Some operator docs reference
   premium flows.
4. CE-specific `LICENSE` file: MIT for everything except trademark? Loop legal in.

## 10. Out of scope

- Closed-source Premium binary distribution (see `license-api-plan.md` §9).
- Air-gap Enterprise bundle.
- Homebrew tap formula updates (separate `homebrew-tap-bootstrap.md`).
- CE-side CI (the CE repo gets no CI of its own — the monorepo is the source of truth).
