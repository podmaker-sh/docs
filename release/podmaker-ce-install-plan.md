# PodMaker CE — Install + First-Boot Plan

> **Status:** draft (2026-06-02). Not started.
> **Owner:** TBD (installer / control-plane).
> **Parent:** [`self-hosted-licensing-plan.md`](./self-hosted-licensing-plan.md) §0a.
> **Sibling plans:** [`podmaker-ce-mirror-plan.md`](./podmaker-ce-mirror-plan.md), [`podmaker-ce-pairing-plan.md`](./podmaker-ce-pairing-plan.md).

CE is delivered as a one-shot installer that turns a fresh VPS / dedicated
box into a running self-hosted PodMaker panel. The operator either pipes
`curl … | sh` on the box or runs the same command over SSH from their own
machine. After the script finishes, the panel is bound to the host's IP
(or a chosen domain) and serves a first-boot setup wizard: pick admin
username + password, then the panel goes live. From that point the
operator can deploy sites + shared services on the install host or on
additional VPSes attached through the panel.

## 0. Kickoff prompt (new session)

> Read `docs/release/podmaker-ce-install-plan.md` and build §3–§5: the
> `get.podmaker.sh/ce` installer script (`scripts/installer/ce-install.sh`),
> the systemd unit templates under `deploy/ce/systemd/`, and the Laravel
> first-boot wizard under `apps/control-plane/app/Filament/Pages/Setup/`.
> Pairing (§6) is handled by `podmaker-ce-pairing-plan.md` — leave the
> hook stubs in `FirstBootCompleted` event listeners and stop. Open
> questions in §10 are blocking.

## 1. Goals + non-goals

| Goal | Why |
|---|---|
| Single command on a fresh box → live panel in < 5 min | Friction-free CE adoption; SSH-exec form keeps it scriptable |
| First-boot wizard for admin credentials only — no choices the script can default | Most operators bounce on questionnaires; default everything else |
| Idempotent re-runs | Operator can re-execute the installer to repair without nuking data |
| systemd-supervised services (CP, controllers, agent, Bao) | One reboot = one healthy panel; no docker-compose-only path |
| Works on Ubuntu 22.04 + 24.04, Debian 12, Rocky / Alma 9 | Cover ≥ 90% of VPS images |

| Non-goal | Reason |
|---|---|
| Provider provisioning (Terraform / cloud-broker) | CE is single-host (parent §0a) |
| Container runtime selection menu | Use podman by default; docker is a flag |
| Multi-host install from one runner | Attach extra VPSes from the panel after the first one is up |
| Auto-TLS via external DNS API | Caddy + ACME HTTP-01 against the chosen domain; operator handles DNS |

## 2. Install vectors

### 2.1 Pipe-to-shell on the host

```sh
curl -fsSL https://get.podmaker.sh/ce | sh
```

`get.podmaker.sh/ce` is a Caddy-served static file that delivers
`ce-install.sh` from the latest `ce-v*` release on
`podmaker-sh/podmaker-ce`. The script is checksummed against the
release notes; the operator can verify with the alternate form:

```sh
curl -fsSL https://get.podmaker.sh/ce.sh -o ce-install.sh
curl -fsSL https://get.podmaker.sh/ce.sh.sha256 | sha256sum -c
sudo sh ce-install.sh
```

### 2.2 SSH-exec from the operator's workstation

```sh
ssh root@<host> 'curl -fsSL https://get.podmaker.sh/ce | sh'
```

Same script. Variables can be passed inline:

```sh
ssh root@<host> 'PODMAKER_DOMAIN=panel.example.com curl -fsSL https://get.podmaker.sh/ce | sh'
```

### 2.3 Re-run / upgrade

```sh
sudo /opt/podmaker/bin/pdctl ce upgrade
```

Pulls the next `ce-v*` release, runs `ce-install.sh` in `--upgrade`
mode (skips bootstrap, runs migrations, restarts services).

## 3. `scripts/installer/ce-install.sh`

Single bash file, no runtime deps beyond a POSIX-ish shell + `curl`.
Phases:

1. **Detect** OS + arch + container runtime + IP / domain.
2. **Confirm** (skipped if `PODMAKER_YES=1`).
3. **Install OS packages** — `podman` (or `docker` via flag),
   `caddy`, `sqlite3`, `git`, `jq`, `openssh-server`.
4. **Create user + dirs** — `podmaker` system user, `/opt/podmaker`,
   `/var/lib/podmaker`, `/etc/podmaker`.
5. **Fetch CE release tarball** from
   `github.com/podmaker-sh/podmaker-ce/releases/latest` — verify cosign
   signature against the embedded public key.
6. **Provision local Bao** via `apps/vault-broker` initd helper —
   single-node, sealed with a key written to
   `/etc/podmaker/bao-unseal.key` (root-only).
7. **Write configs** — `/etc/podmaker/ce.env` with discovered IP +
   chosen domain (defaults to `https://<ipv4>.nip.io` so HTTPS works
   without a real DNS record).
8. **systemd units** — see §4.
9. **Generate install identity** — call out to the pairing plan
   (§6) to mint the Ed25519 keypair + `install_uuid`. Persisted in
   `/etc/podmaker/install.id` (UUID) and `/etc/podmaker/install.key`
   (private key, root-only).
10. **Run migrations** — `php artisan migrate --force` inside the CP
    container.
11. **First-boot marker** — writes `/var/lib/podmaker/first-boot.flag`
    if not present; the wizard (§5) reads this to decide whether to
    show or skip.
12. **Print** the panel URL + setup wizard URL + log file paths.

Exit codes:
- `0` — clean install / upgrade
- `10` — OS not supported
- `11` — port 80/443 already bound by something else
- `12` — pairing handshake failed (advisory — install continues; the
  CE panel is usable, license features remain locked)

Environment variables (all optional, all `PODMAKER_*` prefix):

| Var | Default | Meaning |
|---|---|---|
| `PODMAKER_DOMAIN` | `<ipv4>.nip.io` | bind hostname; controls Caddy + ACME |
| `PODMAKER_IP` | autodetect | bind IP if multi-NIC |
| `PODMAKER_RUNTIME` | `podman` | `podman` or `docker` |
| `PODMAKER_CHANNEL` | `stable` | `stable` / `edge` — picks release tag prefix |
| `PODMAKER_YES` | unset | non-interactive (skip confirms) |
| `PODMAKER_PAIR` | `1` | set to `0` to skip the `podmaker.sh` pairing handshake (offline install) |

## 4. systemd units (`deploy/ce/systemd/`)

Each unit runs the named binary or container under the `podmaker`
user. `Restart=on-failure`, `RestartSec=5s`. Dependencies are
expressed via `After=` + `Requires=`.

| Unit | Binds |
|---|---|
| `podmaker-cp.service` | control-plane (Laravel + Octane) on `127.0.0.1:8080` |
| `podmaker-caddy.service` | Caddy fronting CP + the wizard on `:443` |
| `podmaker-orchestrator.service` | orchestrator on `127.0.0.1:9000` |
| `podmaker-agent-gateway.service` | agent-gateway on `:7700` for attached VPSes |
| `podmaker-controller-*.service` | one unit per controller in the CE allowlist |
| `podmaker-bao.service` | local Bao single-node on `127.0.0.1:8200` |
| `podmaker-pairing.timer` | calls `pdctl ce pair sync` every 1h |

The Caddy file (`/etc/podmaker/Caddyfile`) auto-renews TLS for the
chosen domain. For `*.nip.io` it uses staging by default and the
operator is prompted to switch to prod after picking a real domain.

## 5. First-boot setup wizard

Lives at `apps/control-plane/app/Filament/Pages/Setup/`. Only
accessible while `/var/lib/podmaker/first-boot.flag` exists. Steps:

1. **Welcome** — show install URL + install UUID + pairing status
   (paired / unpaired, link to `panel.podmaker.sh`).
2. **Admin account** — username (default `admin`), email, password
   (≥ 12 chars), 2FA prompt (skippable).
3. **Domain confirm** — re-show detected domain; let the operator
   change it if they want to point a real DNS record now.
4. **Optional: pair with podmaker.sh** — if `PODMAKER_PAIR=0` was set
   or the handshake failed, show a button + manual token form.
   Otherwise show "already paired as `install_<uuid>`" + link to the
   membership page on `panel.podmaker.sh`.
5. **Finish** — drop the first-boot flag, redirect to the panel home.

The wizard refuses to finish if the admin password is unset or the
install identity is missing. It does **not** require pairing — CE
runs locked to OSS features without a license, and the operator can
pair later from Settings → Licensing.

## 6. Pairing handshake (sketch — full spec in pairing plan)

During install step 9:

1. Generate Ed25519 keypair locally.
2. POST `https://api.podmaker.sh/v1/ce/installs/register`
   with `{ install_uuid, public_key, host_fingerprint, cp_version, install_channel }`.
3. Receive `{ paired: true, registration_id, server_pubkey, announcements_topic }`.
4. Persist to `/etc/podmaker/install.id` + `/etc/podmaker/install.pair`.
5. From this point the panel can pull announcements (parent §0a
   dashboard widget — see pairing plan §3 for the endpoint) and
   later receive a license JWT after the operator buys via
   `panel.podmaker.sh`.

If the handshake fails (offline install, network glitch), the
install completes anyway and `pdctl ce pair` can be re-run later.

## 7. Filesystem layout

```
/opt/podmaker/
  bin/                    # pdctl + helper scripts
  releases/               # versioned CE tarballs (last 3 kept)
  current/                # symlink to active release
/etc/podmaker/
  ce.env                  # bind IP, domain, runtime, channel
  Caddyfile               # TLS termination
  install.id              # install UUID (mode 0600)
  install.key             # Ed25519 private key (mode 0600)
  install.pair            # paired-server pubkey + registration_id
  bao-unseal.key          # root-only
/var/lib/podmaker/
  cp-storage/             # Laravel storage/
  bao-data/
  pg-data/                # local PostgreSQL data
  agent-inventory/        # attached-VPS metadata
  first-boot.flag         # absent after wizard completes
/var/log/podmaker/
  install.log
  *.log                   # one per service
```

## 8. pdctl CE subcommands

The mirror denylists `cmd/podmakerctl/license.go` but ships a parallel
`cmd/podmakerctl/ce.go` that is OSS-safe:

- `pdctl ce upgrade` — pull next release + restart.
- `pdctl ce pair` — re-run the pairing handshake (or pair for the first
  time if offline-installed).
- `pdctl ce status` — install UUID, paired-as, license state, services
  health, last announcement fetch.
- `pdctl ce attach-vps <ip> --user <user> --key <ssh-key-path>` —
  enroll an additional VPS into the agent inventory; runs the
  agent installer over SSH.
- `pdctl ce backup` — snapshot pg-data + bao-data + cp-storage to a
  tarball.

Premium subcommands (`pdctl license install`, etc.) live in
`cmd/podmakerctl/license.go` — denylisted, not present in the CE
build, attempts return "this subcommand is part of PodMaker Premium".

## 9. Bootstrap (release engineering)

1. Stand up `get.podmaker.sh` (Caddy on the existing marketing host or
   a dedicated `gh-pages` redirect — TBD §10.Q1).
2. CI step in `publish-podmaker-ce.yml`: after the mirror commit, also
   publish the `ce-install.sh` tarball + checksums as a GitHub release
   asset on `podmaker-sh/podmaker-ce`.
3. Cosign-sign the installer script with the existing CE release key
   (separate from the license JWT key, mirror the manifestsig pattern).

## 10. Open questions

1. `get.podmaker.sh` host — reuse marketing host, GitHub Pages
   redirect, or a dedicated tiny Caddy box? Affects DNS + SLA.
2. Default container runtime — `podman` (rootless-friendly, modern
   default) vs `docker` (familiar)? The plan picks podman; confirm.
3. First-boot wizard skin — the existing panel theme or a stripped
   single-page Tailwind shell?
4. `*.nip.io` HTTPS — staging cert acceptable for first 24h? Operator
   may not understand the browser warning. Maybe ship a "click here
   to trust local CA" path?
5. Database — SQLite for the first install, with an "upgrade to
   local PostgreSQL" wizard step, vs PostgreSQL always-on from
   minute one? PostgreSQL adds ~150MB RAM idle.
6. Telemetry on install failure — opt-in send of `install.log` tail
   to `api.podmaker.sh` for debugging?

## 11. Out of scope

- Provider-side VPS provisioning (CE attaches existing VPSes).
- Kubernetes packaging (CE is single-host + attached VPSes; clusters
  are SaaS).
- Multi-region or HA topology of the CE panel itself.
- Anti-tamper of `install.key` beyond filesystem perms.
