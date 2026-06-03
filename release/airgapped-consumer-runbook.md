# Air-Gapped Consumer Runbook

How to pull, verify, and operate the podmaker orchestrator image
without network access to `ghcr.io` or `raw.githubusercontent.com`
at deploy time.

> Companion doc: `docs/release/manifest-signing-rotation.md` — the
> orchestrator-side signing pipeline. This runbook is the
> consumer-side mirror.

## 1. What gets carried across the air gap

Three artefacts. Pin all three to the same release tag:

| Artefact | Source | Verified by |
|---|---|---|
| Orchestrator container image | `ghcr.io/podmaker-sh/orchestrator:vX.Y.Z` | OCI labels (§3) |
| Signed kinds manifest | `artifacts/kinds-manifest` branch, file `artifacts/kinds-manifest.json` | Ed25519 signature (§4) |
| pdctl binary with embedded trust ledger | `github.com/podmaker-sh/podmaker/releases/download/vX.Y.Z/pdctl-<os>-<arch>` | Reproducible build hash (§5) |

The trust ledger inside pdctl pins the keys that can have signed
the kinds manifest. Updating pdctl is the only way the consumer
picks up a rotated signer (see rotation doc §3 grace period).

## 2. Pull on the network-connected staging box

```bash
TAG=v0.4.4

# Image — multi-arch (amd64 + arm64).
docker pull ghcr.io/podmaker-sh/orchestrator:${TAG}

# Signed manifest — branch HEAD or pinned commit.
curl -fsSL -o kinds-manifest.json \
    "https://raw.githubusercontent.com/podmaker-sh/podmaker/artifacts/kinds-manifest/artifacts/kinds-manifest.json"

# pdctl — pick the right arch.
curl -fsSL -o pdctl \
    "https://github.com/podmaker-sh/podmaker/releases/download/${TAG}/pdctl-linux-amd64"
chmod +x pdctl
```

## 3. Verify image-to-manifest correlation

The orchestrator image carries the manifest's sha + key_id + count
as OCI labels. Refuse to ship a mismatch:

```bash
EXPECTED_SHA=$(sha256sum kinds-manifest.json | cut -d' ' -f1)
GOT_SHA=$(docker inspect ghcr.io/podmaker-sh/orchestrator:${TAG} \
    --format '{{ index .Config.Labels "org.podmaker.kinds_manifest_sha" }}')

if [ "$EXPECTED_SHA" != "$GOT_SHA" ]; then
    echo "manifest sha mismatch — image ($GOT_SHA) vs file ($EXPECTED_SHA)" >&2
    exit 1
fi

# Sanity: count > 0, key_id labelled.
docker inspect ghcr.io/podmaker-sh/orchestrator:${TAG} \
    --format '{{ index .Config.Labels "org.podmaker.kinds_manifest_count" }}'
docker inspect ghcr.io/podmaker-sh/orchestrator:${TAG} \
    --format '{{ index .Config.Labels "org.podmaker.kinds_manifest_key_id" }}'
```

A mismatch means the image was built before the latest manifest
landed (or vice versa); refuse to ship until both come from the
same release.

## 4. Verify the manifest signature

`pdctl` carries the trust ledger in its binary, so no extra key
material crosses the gap:

```bash
./pdctl tier validate t2-managed-aws \
    --against-registry ./kinds-manifest.json \
    --from t1-single-host-aws
# Output should include:
#   Signature    : verified (key_id=…, embedded ledger)
#   Kind check   : OK (every plan step has a registered Apply)
```

If `Signature` says `present but unverified` — your pdctl is too
old. Pull a newer release.

If `Kind check` says `FAILED — missing in registry` — your tier
YAML uses a plan step the orchestrator image you're about to deploy
doesn't know how to apply. Either:

- Update the tier YAML to remove the unsupported step, OR
- Pin the orchestrator image to a release that exposes the step.

## 5. Transfer to the air-gapped network

Use whatever your org's transfer mechanism is (sneakernet, S3
gateway, IronKey, …). Carry the three artefacts together as a
single tarball so the receiver can re-run the verification:

```bash
TAG=v0.4.4
mkdir podmaker-${TAG}-airgap
docker save ghcr.io/podmaker-sh/orchestrator:${TAG} | gzip \
    > podmaker-${TAG}-airgap/orchestrator.tar.gz
cp kinds-manifest.json pdctl podmaker-${TAG}-airgap/
tar czf podmaker-${TAG}-airgap.tar.gz podmaker-${TAG}-airgap/
```

On the air-gapped side:

```bash
tar xzf podmaker-${TAG}-airgap.tar.gz
cd podmaker-${TAG}-airgap
gunzip -c orchestrator.tar.gz | docker load
# Repeat §3 + §4 against the loaded image + carried files.
```

## 6. Re-verification cadence

Every time the orchestrator image rolls (security patch, new
action kind, …) the consumer MUST repeat the steps in §2-§5. The
sha label tying image to manifest only protects intra-release —
mismatch detection across releases requires the operator to pin
the tag they're shipping.

## 7. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `Signature : verified ... but Kind check FAILED` | Image rolled forward, manifest didn't (or vice versa) | Re-pull both at the same `TAG` |
| `Signature : did not match any trusted key` | Old pdctl + rotated signer | Pull the latest pdctl (it embeds the new key) |
| `manifest sha mismatch` | Image label doesn't match the file sha | Same as above — same release on both halves |
| `Signature : present but unverified` | pdctl predates the trusted-ledger feature (< v0.4.4) | Upgrade pdctl |
| `Kind check : missing ProvisionXxx` | Tier YAML uses an action your orchestrator doesn't ship | Bump orchestrator image or trim the YAML |

## 8. Related docs

- `docs/release/manifest-signing-rotation.md` — orchestrator-side
  signing pipeline.
- `packages/shared-go/pkg/manifestsig/` — code.
- `apps/orchestrator/cmd/kinds-dump/` — manifest emitter.
