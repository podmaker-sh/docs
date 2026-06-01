# Sprint 4 — E2E Demo Runbook

Goal: end-to-end deploy of a static site to a fresh Hetzner cx21 with
HTTPS, driven entirely by `pdctl up`, in under six minutes from a
cold start.

## Prereqs

- Docker daemon running (`make dev-up` orchestrates Postgres + Redis +
  OpenBao + Temporal + step-ca + NATS).
- Working hcloud account + an `HCLOUD_TOKEN` with Read+Write scope.
- Working Cloudflare account + an API token with Zone.DNS read+write.
- A registered DNS apex (e.g. `demo.podmaker.io`) pointing its
  nameservers at Cloudflare.

## Phase 1 — Bring up the stack

```sh
make dev-up                                  # Postgres, Redis, Vault, Temporal, ...
cd apps/control-plane
php artisan migrate
php artisan db:seed                          # DatabaseSeeder + DemoSeeder
php artisan serve --host=127.0.0.1 --port=8000 &
```

In a second terminal, start the orchestrator + agent-gateway + vault-
broker:

```sh
ACME_ACCEPT_TOS=true ACME_DIRECTORY_URL=http://127.0.0.1:14000/dir \
  TEMPORAL_HOST_PORT=127.0.0.1:7233 \
  CONTROL_PLANE_URL=http://127.0.0.1:8000 \
  PODMAKER_INTERNAL_TOKEN=podmaker-dev-internal \
  go run ./apps/orchestrator/cmd/orchestrator &

PODMAKER_INTERNAL_TOKEN=podmaker-dev-internal \
  go run ./apps/agent-gateway/cmd/agent-gateway &

PODMAKER_INTERNAL_TOKEN=podmaker-dev-internal \
  go run ./apps/vault-broker/cmd/vault-broker &
```

Optional (Pebble for ACME):

```sh
docker compose -f docker-compose.dev.yml --profile acme up -d pebble
```

## Phase 2 — Operator log in

```sh
make build-ctl                                              # builds bin/podmakerctl
PAT=$(php artisan tinker --execute="echo \\App\\Models\\User::where('email','owner@podmaker.test')->first()->createToken('demo')->plainTextToken;")
./bin/podmakerctl login --url http://127.0.0.1:8000 --token "$PAT"
./bin/podmakerctl whoami    # expect: owner@podmaker.test
```

## Phase 3 — Link provider + provision server

```sh
./bin/podmakerctl provider add hetzner --token "$HCLOUD_TOKEN" --name "Demo Hetzner"
./bin/podmakerctl server add --name demo-01 --provider hetzner --type cx11 --region nbg1
# ~60-90s; agent enrolls + hellos
./bin/podmakerctl server list   # status should advance pending -> provisioning -> active
```

Expected audit events (check `/admin/audit-logs`):

- `cloud_account.linked`
- `server.provision.queued`
- `server.provision.started`
- `server.provision.succeeded`
- `agent.enrolled`

## Phase 4 — Deploy

```sh
./bin/podmakerctl validate docs/demo/setup.yml          # offline schema check
./bin/podmakerctl up -f docs/demo/setup.yml --follow
```

Expected workflow steps stream (in order):

1. acquire_lease
2. upsert_dns
3. issue_cert
4. ensure_firewall
5. build
6. deploy_to_target
7. switch_traffic
8. health_check
9. finalize
10. revoke_lease

Acceptance check:

```sh
curl -I https://hello.demo.podmaker.io
# HTTP/2 200, valid TLS cert chain
```

## Phase 5 — Rollback

Make a trivial change (commit `bad` to the repo, force a failure) and
deploy again. The workflow's compensation should walk back through
revert_traffic + revoke_cert + delete_dns_record + revoke_lease and the
release row stays at `failed`. Then:

```sh
./bin/podmakerctl rollback --site <site-id>
```

Audit events:

- `site.release.queued` (the bad deploy)
- `site.release.finalized` (outcome=failure)
- `site.rollback`
- `site.release.finalized` (outcome=success for the rollback release)

## Acceptance signals

- [ ] First deploy < 6 minutes wall-clock.
- [ ] Second `pdctl up` of an unchanged manifest < 90 seconds.
- [ ] `curl https://hello.demo.podmaker.io` returns 200 with valid TLS.
- [ ] AuditLog tab shows every step listed above with no `outcome=failure`.
- [ ] Demo Friday video recorded and stored under `docs/demo/recordings/`.

## Teardown

```sh
./bin/podmakerctl server destroy <id>
make dev-down
```
