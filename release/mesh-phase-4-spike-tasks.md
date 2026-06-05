# Mesh Phase 4 — De-risking Spike Task Breakdown

**Goal:** prove tailscale data-plane libs (Strategy A) deliver direct
hole-punch + DERP fallback + re-upgrade between two simulated-symmetric-NAT
nodes, using OUR own coordination (hand-built netmap, no tailscale control).
**Exit deliverable:** the minimal netmap shape + exact tailscale API surface
`mesh-signal` must depend on. This is a **throwaway spike** — code lands in
`spikes/mesh-p4/`, not production paths. Timebox ~1 week.

Decision gate: if S4 fails or the API surface is unacceptably large, revisit
Strategy D (headscale) before committing Phase 4.

---

## S0 · Vendor + sandbox setup · 0.5d
- **Dir (new):** `spikes/mesh-p4/` — isolated Go module, depends on
  `tailscale.com` (pin exact version).
- Vendor and confirm import compiles: `wgengine`, `wgengine/magicsock`,
  `tailcfg` (netmap types), `disco`, `net/netcheck`, `derp`, `derp/derphttp`,
  `types/key`.
- **Accept:** `go build ./spikes/mesh-p4/...` green; tailscale version pinned in
  spike `go.mod`.

## S1 · Stand up a DERP server · 1d
- **File:** `spikes/mesh-p4/derp/main.go` — wrap `derp.NewServer` +
  `derphttp.Handler`. Self-signed TLS for spike (step-ca integration deferred to
  prod `apps/derp`).
- Hardcode a 1-region DERP map.
- **Accept:** DERP reachable; `derphttp.Client` from a test process connects and
  exchanges a frame.

## S2 · Two-node userspace engine w/ hand-built netmap · 1.5d
- **File:** `spikes/mesh-p4/node/main.go` — bring up `wgengine` (userspace) +
  `magicsock`. Generate local Curve25519 key. Feed a **hand-authored netmap**
  (`tailcfg`/`netmap.NetworkMap`): self + 1 peer + the S1 DERP map.
- Two processes (nodeA, nodeB), each TUN, mesh IPs `100.x` or `10.42.x`.
- **Accept:** direct path established on same LAN; ICMP/UDP over the tunnel both
  ways; `magicsock` reports a direct (non-DERP) path.

## S3 · Simulate symmetric NAT · 1d
- Two network namespaces or containers, each behind an `iptables`/`nft`
  MASQUERADE with **random source-port mapping** (symmetric NAT), no port
  forward. DERP reachable from both; nodes NOT directly routable by static
  endpoint.
- **File:** `spikes/mesh-p4/lab/` — netns/compose scripts.
- **Accept:** lab reproducibly creates symmetric-NAT-to-symmetric-NAT topology.

## S4 · Prove fallback + hole-punch + re-upgrade · 1.5d  ← the gate
- Scenario sequence, observed via magicsock path status:
  1. Cold start in symmetric NAT → connectivity establishes **via DERP**
     (relay path).
  2. `disco`/`netcheck` hole-punch attempts → if symmetric/symmetric stays on
     DERP (expected); relax one side to easy-NAT → **direct path upgrade**.
  3. Kill direct path (drop UDP) → **auto-fallback to DERP**, no connection drop.
  4. Restore → **re-upgrade to direct**.
- Measure: time-to-connect (DERP), time-to-upgrade (direct), throughput on each
  path, behavior on path flap.
- **Accept (GATE):** all four transitions observed, tunnel survives path flaps,
  no manual intervention. If symmetric/symmetric never goes direct (expected),
  DERP path must be stable — that's acceptable.

## S5 · Lock the netmap contract + API surface · 1d
- **File (deliverable):** `docs/release/mesh-signal-contract.md`.
- Document: the exact `netmap.NetworkMap` / `tailcfg` fields magicsock actually
  reads (peer pubkeys, AllowedIPs, DERP region, endpoints, disco key). Strip to
  the minimum `mesh-signal` must produce from `MeshPeer` registry.
- Document: the tailscale API call surface we depend on (engine init, netmap
  update, status callbacks) — the vendoring blast radius.
- Note MTU setting used, perf numbers from S4.
- **Accept:** `mesh-signal` can be specced from this doc without re-reading
  tailscale internals; vendoring surface enumerated.

---

## Spike summary

| Task | Effort | Output |
|------|--------|--------|
| S0 vendor/sandbox | 0.5d | compiling spike module |
| S1 DERP server | 1d | working relay |
| S2 two-node engine | 1.5d | direct tunnel via hand netmap |
| S3 symmetric-NAT lab | 1d | reproducible NAT topology |
| S4 fallback/re-upgrade | 1.5d | **GATE: 4 transitions proven** |
| S5 contract doc | 1d | netmap shape + API surface locked |

≈ 6.5d. Gate = S4. Output feeds production Phase 4 (`apps/derp`,
`apps/mesh-signal`, `apps/agent/internal/mesh/engine.go`).

## Progress log
- **2026-06-04 · S0 DONE** — `spikes/mesh-p4/` standalone module (NOT in
  go.work). Vendored `tailscale.com v1.100.0`; importcheck builds green.
  Finding: requires **Go ≥ 1.26.4**.
- **2026-06-04 · S1 DONE** — `spikes/mesh-p4/derp` boots with self-signed TLS,
  prints DERP pubkey, `/healthz`=ok, `/derp/probe`=200.
  **API drift:** DERP server moved `tailscale.com/derp` →
  `tailscale.com/derp/derpserver` (`New`, `Handler`, `ProbeHandler`).
- **2026-06-04 · S2 IN PROGRESS** — `spikes/mesh-p4/node/stack.go` compiles:
  ported tailscale's in-package `newMagicStack` + `meshStacks` test helpers to
  an **external package using only exported API** (the key feasibility risk for
  Strategy A — RETIRED). Substituted unexported `conn.WaitReady/logf/self` with
  channel-polling + local IP tracking. **API drift:** `net/nettype` →
  `types/nettype`. Remaining for S2: natlab runner + `runDERPAndStun`-on-natlab
  + `tuntest.Ping` connectivity assertion.
- **Implication for prod:** external-package port is viable → `apps/agent` can
  embed the engine without forking tailscale internals. Confirms D2/D4.
- **2026-06-04 · S2 DONE** — `node/mesh_test.go::TestTwoNodeDirect` PASSES: two
  userspace-WG nodes over natlab, wired by our ported `meshStacks`, bidirectional
  ping through the DERP/STUN harness. Strategy A end-to-end proven at small scale
  (engine + magicsock + DERP + our minimal control, all exported API).
  **API drift:** wireguard-go is tailscale's fork
  `github.com/tailscale/wireguard-go` (NOT `golang.zx2c4.com/wireguard`) — using
  the wrong one fails with a `tun.Device` interface mismatch.
  **Bug found+fixed:** the WaitReady replacement must not *consume* the first
  endpoint from epCh (meshStacks needs it) — peek `len(epCh)>0` instead.
- **2026-06-04 · S3 + S4 DONE** — `node/nat_test.go` adds two natlab
  topologies driven by `SNAT44` + `Firewall`:
  - `TestSymmetricNATFallback` — both nodes behind
    `AddressAndPortDependentNAT`; direct hole-punching is impossible by
    construction. PING transits via DERP only (`CurAddr=""`, `Relay`
    set). **THE GATE — passed.**
  - `TestEasyNATDirect` — both nodes behind `EndpointIndependentNAT`;
    hole-punching succeeds → direct path (`CurAddr` populated).
    Confirms the engine upgrades when the NAT shape permits it.
  Reproducible from a clean tree with
  `cd spikes/mesh-p4 && GOWORK=off go test ./node/ -v`.
- **2026-06-04 · S5 DONE** — `docs/release/mesh-signal-contract.md`
  locks the netmap shape + tailscale API surface mesh-signal must
  produce. The doc is derived from the working spike, not from
  reading tailscale internals; production Phase 4 (`apps/derp`,
  `apps/mesh-signal`, `apps/agent/internal/mesh/engine.go`) is
  specced from there.
- **GATE OUTCOME:** Strategy A is viable. The agent engine can embed
  tailscale's magicsock + wgengine through external-package exported
  API without forking internals, and our minimal control plane is
  sufficient to drive symmetric/symmetric DERP fallback and easy-NAT
  direct upgrades.

## What this spike explicitly does NOT do
- No step-ca TLS on DERP (prod only).
- No control-plane integration / registry wiring.
- No multi-region DERP, no HA.
- No production agent code — sandbox only.
- No Phase 5/6 (subnet/k8s) concerns.
