# vault-bridge release pipeline secrets

The `release-vault-bridge.yml` workflow needs five secrets configured
on the `podmaker-sh/podmaker` (private monorepo) repo. None of them
are committed to the working tree; they live in the repo's
**Settings → Secrets and variables → Actions** pane.

## `RELEASES_PAT` — cross-repo binary publish

Fine-grained PAT, owned by `podmaker-bot`. Scope:

| Setting             | Value                              |
|---------------------|------------------------------------|
| Resource owner      | `podmaker-sh`                      |
| Repository access   | Only **`podmaker-sh/releases`**    |
| Repository perms    | Contents: **Read and write**       |
| Expiration          | 1 year                             |

```sh
gh secret set RELEASES_PAT --repo podmaker-sh/podmaker --body "<paste>"
```

The `binaries`, `packages`, and `msi` jobs use this to upload
tarballs / .deb / .rpm / MSI assets to the public
`podmaker-sh/releases` repo — the monorepo stays private.

## `DOCS_PAT` — cross-repo docs publish (optional)

Same shape as `RELEASES_PAT`, scoped to `podmaker-sh/docs`. The
`publish-docs.yml` workflow uses it to mirror customer-facing
docs out of the monorepo on a daily cron.

## `HOMEBREW_TAP_PAT` — homebrew tap auto-bump

Fine-grained Personal Access Token, owned by the `podmaker-bot`
machine user. Scope:

| Setting              | Value                                |
|----------------------|--------------------------------------|
| Resource owner       | `podmaker-sh` (the org)              |
| Repository access    | Only **`podmaker-sh/homebrew-tap`**    |
| Repository perms     | Contents: **Read and write**         |
| Repository perms     | Metadata: Read-only (auto)           |
| Account permissions  | none                                 |
| Expiration           | 1 year, calendar reminder to rotate  |

Steps:

1. Sign in as `podmaker-bot`. Settings → Developer settings →
   Personal access tokens → Fine-grained tokens → Generate new.
2. Apply the table above and create the token.
3. On `podmaker-sh/podmaker`: Settings → Secrets → Actions → New
   repository secret. Name `HOMEBREW_TAP_PAT`, paste the value.

The `brew-tap` job checks the tap out with this token, rewrites
`Formula/podmaker-vault-bridge*.rb`, and pushes a single commit
per release. If the tap repo does not exist yet, create
`podmaker-sh/homebrew-tap` with a single `Formula/` directory before
running the workflow.

## `WINDOWS_CODESIGN_CERT_B64` + `WINDOWS_CODESIGN_CERT_PASS` — MSI signing

The MSI job signs `podmaker-vault-bridge-<version>-x64.msi` with
`signtool.exe` before uploading it to the release. Without these
secrets the job emits an unsigned MSI and the Release notes warn
the customer that SmartScreen will block silent install.

| Secret                     | Format                                                 |
|----------------------------|--------------------------------------------------------|
| `WINDOWS_CODESIGN_CERT_B64`| Base64 of the `.pfx` (`base64 cert.pfx > cert.b64`)    |
| `WINDOWS_CODESIGN_CERT_PASS`| Plaintext PFX export password                         |

EV certs are preferred — they bypass SmartScreen reputation warm-up
that costs ~1 month on an OV cert. The PFX must include the full
chain. Renew via the issuer's portal before the cert expires;
unsigned MSIs are accepted by `msiexec /qn` but rejected by Group
Policy on most domain-joined machines.

## `COSIGN_REGISTRY_OIDC` — implicit, no secret

cosign keyless OIDC uses the workflow's `id-token: write` permission
plus the GitHub OIDC issuer. No secret needs to be configured —
just leave `id-token: write` in the workflow YAML. The cert is
pinned to `https://github.com/<org>/<repo>/.github/workflows/...`
so a fork cannot mint a sibling signature.

## Rotation policy

- `HOMEBREW_TAP_PAT` — 1 year, manual rotate, calendar reminder.
- `WINDOWS_CODESIGN_CERT_*` — match cert validity (typically 1y EV).
- cosign trust roots — automatic via `cosign-installer@v3`.

## Verification

After secrets land, dispatch the workflow manually:

```
gh workflow run release-vault-bridge.yml -f version=v0.0.0-secret-check
```

Expect every job (`binaries`, `msi`, `brew-tap`, `image`) to succeed.
A failing `brew-tap` is the most common — usually the PAT does not
have write access to the tap repo, or the tap repo doesn't exist yet.
