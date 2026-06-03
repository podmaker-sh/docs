# PodMaker CE — Release Process Runbook

> **Status:** active (2026-06-03). First cosign-signed release ships
> in CE v0.1.0.
> **Owner:** Release engineering.
> **Parent:** [`podmaker-ce-mirror-plan.md`](./podmaker-ce-mirror-plan.md),
> [`podmaker-ce-install-plan.md`](./podmaker-ce-install-plan.md).
> **Sibling plans:** [`manifest-signing-rotation.md`](./manifest-signing-rotation.md)
> (rotation pattern this doc mirrors).

This runbook covers cutting a `ce-vX.Y.Z` release: tag → mirror sync →
release bundle build → cosign signing → GitHub release publish →
`get.podmaker.sh` redirector flips automatically.

## 0. One-time bootstrap (already done as of 2026-06-03)

Run this checklist once per cosign key generation; rotation goes
through §6.

1. Generate the CE release cosign key offline:
   ```sh
   cosign generate-key-pair
   # cosign.key — sealed into 1Password "PodMaker / CE / cosign"
   # cosign.pub — committed to deploy/ce/cosign.pub
   ```
2. Drop `cosign.pub` into `deploy/ce/cosign.pub` (overwriting the
   sprint 15 placeholder).
3. Add GitHub Actions repository secrets to the monorepo:
   - `CE_COSIGN_KEY` — `base64 -w0 < cosign.key`
   - `CE_COSIGN_PUBLIC_KEY` — `base64 -w0 < cosign.pub`
   - `CE_COSIGN_PASSWORD` — passphrase entered during `generate-key-pair`
   - `PODMAKER_CE_PAT` — fine-grained PAT, `contents:write` on
     `podmaker-sh/podmaker-ce` only
4. Stand up `get.podmaker.sh` from
   [`deploy/get-podmaker-sh/Caddyfile`](../../deploy/get-podmaker-sh/Caddyfile).
   Point an A record at the host. Caddy provisions TLS on first
   request.
5. Verify the redirector:
   ```sh
   curl -fsSLI https://get.podmaker.sh/ce          # 302 → github.com
   curl -fsSL  https://get.podmaker.sh/ce.sh       # ce-install.sh body
   curl -fsSL  https://get.podmaker.sh/cosign.pub  # cosign public key
   ```

## 1. Cutting a release

1. Land all CE-bound changes on `main`. Verify locally:
   ```sh
   scripts/release/ce-strip.sh "$PWD" /tmp/ce-out --dry-run
   ```
   Leak guard must pass.
2. Decide the next tag — semver patch for fixes, minor for new
   premium catalogue entries, major for breaking pairing protocol
   changes:
   ```sh
   NEXT_TAG=ce-v0.1.0
   ```
3. Write release notes:
   ```sh
   ${EDITOR:-vim} docs/release/CE-CHANGELOG-${NEXT_TAG}.md
   ```
   Use the same `## Highlights / ## Fixes / ## Premium catalogue`
   sections as the previous tag's file. If absent, the release
   workflow falls back to the auto-generated commit list.
4. Commit the changelog. Tag + push:
   ```sh
   git tag -a "$NEXT_TAG" -m "PodMaker CE $NEXT_TAG"
   git push origin "$NEXT_TAG"
   ```

## 2. What CI does automatically

Pushing the tag triggers two workflows in parallel:

| Workflow | Purpose |
|---|---|
| [`publish-podmaker-ce.yml`](../../.github/workflows/publish-podmaker-ce.yml) | Mirrors stripped source tree to `podmaker-sh/podmaker-ce`, commits with the tag note |
| [`release-podmaker-ce.yml`](../../.github/workflows/release-podmaker-ce.yml) | Builds the release bundle, cosign-signs the installer + tarball + manifest, creates the GitHub release with assets |

Release-bundle pipeline steps (see
[`scripts/release/ce-release-bundle.sh`](../../scripts/release/ce-release-bundle.sh)):

1. Run `ce-strip.sh` against the monorepo to produce the staged
   source tree (same content the mirror workflow pushes).
2. Tarball it as `podmaker-ce-${TAG}.tar.gz`.
3. Copy `scripts/installer/ce-install.sh` verbatim so the curl-pipe
   one-liner works without unpacking the tarball.
4. Emit `manifest.json` (tag + monorepo SHA + sha256 of artefacts).
5. Generate `SHA256SUMS` over all artefacts.
6. Build `RELEASE_NOTES.md` from
   `docs/release/CE-CHANGELOG-${TAG}.md` when present, otherwise
   `git log` since the previous `ce-v*` tag.
7. `cosign sign-blob` against the tarball, installer, and manifest
   — emits `.sig` siblings.
8. Upload artefacts to a GitHub release on `podmaker-sh/podmaker-ce`.

## 3. After the release lands

Operator-facing surfaces resolve automatically:

- `https://get.podmaker.sh/ce` → 302 → latest `ce-install.sh` asset
- `curl -fsSL https://get.podmaker.sh/ce | sh` → installer fetches
  `podmaker-ce.tar.gz` + `SHA256SUMS` + `cosign.pub` + `*.sig`
- The installer verifies the tarball against `SHA256SUMS` first,
  then against the cosign signature when `cosign` is installed on
  the operator's host (rollout window keeps this advisory — flip to
  hard-fail in Sprint 15.5 once every CE install ships with cosign).

## 4. Smoke-testing the release

Always run a fresh-install smoke before announcing externally:

```sh
docker run --rm -it ubuntu:24.04 sh -c '
    apt-get update -qq && apt-get install -y -q curl ca-certificates &&
    curl -fsSL https://get.podmaker.sh/ce | PODMAKER_DOMAIN=test.nip.io PODMAKER_YES=1 sh
'
```

Expected: installer phases pass through to `print_summary`, `pdctl
ce status` reports `paired` (or `unpaired` if `PODMAKER_PAIR=0`),
HTTPS endpoint serves the setup wizard.

## 5. Rollback

Two paths depending on severity:

| Trigger | Action |
|---|---|
| Release built from the wrong commit | Delete the GitHub release; re-push the tag after fixing main; CI re-runs |
| Release passed CI but breaks installs | Mark the release as a draft on GitHub. Operators auto-pinned to the previous tag via `pdctl ce upgrade --pin <prev-tag>` while the fix lands |
| Cosign key compromise | Skip to §6 immediately. Treat in-flight installs as untrusted; the unreachable-grace logic from Sprint 10 keeps premium working for the 7-day window while operators re-pair |

## 6. Cosign key rotation

Mirrors the manifest-signing rotation runbook one-for-one — the
embedded `cosign.pub` is the equivalent of the manifest trust
ledger entry. Rotation cadence: quarterly + on incident.

1. Generate the new pair (`cosign generate-key-pair`).
2. Add both old + new public keys to
   [`deploy/ce/cosign.pub`](../../deploy/ce/cosign.pub) concatenated
   in order (old block first, new block second). The installer
   tries each block until one verifies.
3. Update the `CE_COSIGN_KEY` + `CE_COSIGN_PUBLIC_KEY` secrets to
   the new key.
4. Cut a fresh `ce-vX.Y.Z` release signed with the new key.
5. Wait 30 days for installs to roll forward.
6. Drop the old key block from `deploy/ce/cosign.pub` and ship one
   more release.

## 7. Open questions / Sprint 15.5 backlog

1. Cosign keyless (Sigstore OIDC) — the workflow includes
   `id-token: write` so we can flip to keyless without a workflow
   rewrite once we decide whether to bind the identity to the
   GitHub repo or a service account.
2. CE installer should reject unsigned releases (`PODMAKER_ALLOW_UNSIGNED=1`
   opt-out only). Currently advisory while every operator picks up
   cosign on their host.
3. SBOM attachment — `syft packages` against the staged tree,
   upload as `sbom.spdx.json` per release.
4. Verify the published cosign public key matches the bytes
   embedded in `deploy/ce/cosign.pub` on every CI run — drift
   detector cron.

## 8. Out of scope

- Operator-side `cosign` install (covered by per-OS guides under
  `docs/operator/cosign-install.md` — separate runbook).
- Multi-architecture installer (the installer is POSIX-sh + curl;
  arch detection happens at runtime).
- Air-gapped customer installs (covered by
  [`airgapped-consumer-runbook.md`](./airgapped-consumer-runbook.md)).
