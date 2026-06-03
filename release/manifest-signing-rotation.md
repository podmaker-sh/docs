# Manifest Signing — Key Rotation Strategy

> **Scope:** the Ed25519 signing key behind `kinds-dump --signing-key`
> and `pdctl tier validate --registry-pubkey`. The same pattern
> applies to any future signed artefact emitted by the orchestrator
> (release manifests, license bundles, …) — keep this doc as the
> source of truth.

## 1. Identity model

Every signing key has an 8-byte `key_id` (first 8 bytes of the
public key, hex-encoded). It is the operator-facing identifier:

- printed on key generation (`kinds-dump genkeys`)
- embedded in every signed manifest (`key_id` JSON field)
- printed by `pdctl tier validate` when verification succeeds
- referenced in the trust ledger below

A short id is intentional — operators read it in CI logs, not a
key-exchange protocol.

## 2. Trust ledger

The set of public keys considered authoritative right now lives at
`packages/shared-go/pkg/manifestsig/trusted/*.pub`. Each file is
hex-encoded Ed25519, filename matches `key_id`. The package
`manifestsig.TrustedKeys()` reads them via `go:embed`, so the list
is baked into the shipped pdctl binary at build time.

Adding / retiring a key = a normal PR against this directory.

### Current trusted keys (kept in this doc as a reminder; the
### embedded directory is the source of truth):

| Key ID             | Generated | Retired | Purpose |
|--------------------|-----------|---------|---------|
| `0dedf9ae8d0065b7` | 2026-06-02 | — | first prod signer (cutover from the placeholder `8b89122f63ea09e4`) |
| `8b89122f63ea09e4` | 2026-06-02 | 2026-06-02 | retired placeholder — manifests under it no longer verify |

(Update this table in the same PR that adds / removes a pub key.)

## 3. Multi-key trust + grace period

Because we trust an explicit *set* of keys (not just one), rotation
never breaks verification — old signers stay trusted until they're
explicitly removed.

### Rotation procedure (planned, non-incident)

1. **Generate the new key** on the orchestrator's signing host:

   ```bash
   ./kinds-dump genkeys --priv orch-2026Q3-priv.key --pub orch-2026Q3-pub.key
   # Note the key_id printed.
   ```

2. **Commit the public key** to
   `packages/shared-go/pkg/manifestsig/trusted/<key_id>.pub`.

   Update the table in §2.

3. **Cut a pdctl release** with the new pubkey embedded. Operators
   on this version trust both old and new keys.

4. **Wait one minor-release cycle** (4-6 weeks) so the new pdctl
   version is widely deployed.

5. **Cut over the signer**: update the CI workflow's
   `PODMAKER_MANIFEST_SIGNING_KEY` repo secret to the new private
   key. From this point every new manifest is signed by the new
   key; old manifests stay verifiable via the still-trusted old
   pubkey.

6. **Retire the old key** in the next minor release: delete the old
   pub from `trusted/`, update §2 with the retirement date. Old
   manifests no longer verify under the new pdctl, but they're
   pinned to a known commit anyway — air-gapped consumers should
   re-fetch the latest signed manifest.

### Rotation procedure (incident — key compromise)

1. **Immediately rotate the CI secret** (private key on the signing
   host).
2. **Delete the compromised pub from `trusted/` in the same PR
   that ships the new pub**. Skip the grace period.
3. **Cut an emergency patch release** of pdctl + orchestrator.
4. **Force all air-gapped consumers to upgrade** — there's no
   backwards-compat path. Document the incident in a `SECURITY.md`
   entry referencing the compromised key_id so audit logs make
   sense.
5. **Re-sign every manifest in `artifacts/kinds-manifest` branch
   history** so historical verification still works. (Easier than
   it sounds: rebase + re-run kinds-dump, the JSON is small.)

## 4. CI secret management

The GitHub Actions workflow at
`.github/workflows/orchestrator-kinds-manifest.yml` reads the
private key from `PODMAKER_MANIFEST_SIGNING_KEY` (repo secret). The
key never touches the runner's disk outside the ephemeral
`.signing/` directory, which the final step wipes regardless of
outcome.

DO NOT:

- check the private key into the repo
- pass it on a command line (will land in `set -x` traces)
- export it through any env that leaks into the JSON action output

DO:

- rotate the secret value at the same cadence as §3
- rate-limit the workflow (one run per push, no scheduled runs that
  could leak the key on a debug log)

## 5. Trust ledger refresh check

Two-job CI gate at `.github/workflows/manifest-trust-check.yml`:

- `trust-ledger-shape` — every `trusted/*.pub` file is a 32-byte
  hex Ed25519 public key with a filename matching the canonical
  8-byte key_id. Runs on every PR + push that touches the
  directory.
- `manifest-key-id-drift` — when `artifacts/kinds-manifest.json`
  changes, asserts the manifest's `key_id` has a matching
  `trusted/<id>.pub`. Catches mis-rotated CI secrets before
  anyone deploys a manifest no pdctl can verify.

## 6. Operator runbook — first prod key

> Do this once, before the public release.
> The script wraps `kinds-dump genkeys` and prints exact commands
> for the GitHub secret + the trusted-ledger PR.

```bash
./scripts/release/gen-prod-signing-key.sh
```

The script:

1. Mints the key pair into a `mktemp` dir (deleted on exit).
2. Prints the GitHub-CLI invocation to push the private key into
   the `PODMAKER_MANIFEST_SIGNING_KEY` repo secret.
3. Writes the public half into
   `packages/shared-go/pkg/manifestsig/trusted/<key_id>.pub` so
   you can `git add` + open a PR.
4. Reminds you to update the ledger table in §2 + retire the
   placeholder.

**Pushing the secret** — `make set-signing-secret PRIV=<path>`
wraps the `gh secret set` step so the private hex never lands in
your shell history. Pass the path the script wrote the private
key to (you'll have one terminal session to copy it before the
`mktemp` cleanup kicks in):

```bash
make set-signing-secret PRIV=/tmp/scratch/orch-priv.key
```

**After the PR merges** — kick the first signed-manifest run:

```bash
make trigger-kinds-manifest
# or, raw form:
gh workflow run orchestrator-kinds-manifest.yml
```

The workflow fires and commits the signed manifest to
`artifacts/kinds-manifest`. Verify the new key_id is present:

```bash
git fetch origin artifacts/kinds-manifest
git show origin/artifacts/kinds-manifest:artifacts/kinds-manifest.json | jq '.key_id'
```

Should match the `KEY_ID` the script printed.

## 7. Image label propagation

Air-gapped consumers must correlate an orchestrator container image
with the kinds manifest its `/v1/self-deploy/kinds` endpoint will
serve. The build pipeline does this via three OCI labels stamped at
`docker buildx` time:

| Label                                | Value source |
|--------------------------------------|--------------|
| `org.podmaker.kinds_manifest_sha`    | `sha256(artifacts/kinds-manifest.json)` |
| `org.podmaker.kinds_manifest_key_id` | `jq .key_id` |
| `org.podmaker.kinds_manifest_count`  | `jq .count` |

The `build-image-orchestrator` Makefile target reads the manifest
from `artifacts/kinds-manifest.json` (the `artifacts/kinds-manifest`
branch checked out via the CI workflow), computes the three values,
and passes them as `--build-arg` to the orchestrator Dockerfile.
When no manifest exists locally the labels fall back to `unknown` /
`0`.

Consuming the labels on the verifier side:

```bash
docker inspect ghcr.io/podmaker-sh/orchestrator:v0.4.4 \
  --format '{{ index .Config.Labels "org.podmaker.kinds_manifest_sha" }}'
# → 9a3f...

# Pin this sha when downloading the signed manifest for offline
# validation:
curl -fsSL "https://raw.githubusercontent.com/podmaker-sh/podmaker/artifacts/kinds-manifest/artifacts/kinds-manifest.json" \
    | sha256sum
# → must match the image label
```

## 8. Related docs

- `packages/shared-go/pkg/manifestsig/manifestsig.go` — code reference.
- `.github/workflows/orchestrator-kinds-manifest.yml` — signing
  pipeline.
- `docs/release/self-hosted-licensing-plan.md` — the parallel
  signing story for license bundles. Same key shape, separate key
  pair so a license-key compromise doesn't poison the orchestrator
  manifest signer (and vice versa).
