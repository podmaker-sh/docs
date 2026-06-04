# Mesh Overlay — Phase 4 (NAT Traversal / DERP) Feasibility

**Status:** Approved direction
**Date:** 2026-06-04
**Decision:** Strategy **A** — embed Tailscale data-plane libraries; control,
identity (step-ca), and UI stay ours. DERP relay self-hosted (`apps/derp`).

## The core technical fact (why this drives Phase 0a)

Real full-mesh P2P + DERP fallback requires **userspace WireGuard**
(`wireguard-go`) plus a `magicsock`-style multiplexer:

- Kernel WireGuard (`wg-quick`) binds each peer to **one fixed `Endpoint`**. It
  cannot transparently fail over to a relay — the kernel has no "endpoint =
  pubkey, route over direct-UDP-or-DERP" abstraction.
- Tailscale's hole-punching + DERP fallback + live path migration live in
  `magicsock`, which sits under a userspace WireGuard engine.

**Therefore:** Phase 0a's kernel `wg-quick`/`octa0` plan is **revised to
userspace WireGuard** for any node that participates in NAT-traversed P2P.
(Kernel WG remains valid only for always-reachable public nodes / relay-only.)

## Strategy A — what we embed vs build

Tailscale is BSD-3-Clause → vendorable.

**Embed (vendor `tailscale.com/...`):**
- `wgengine` — userspace WireGuard engine
- `wgengine/magicsock` — UDP/DERP multiplexing packet conn
- `disco` — NAT-traversal probe protocol (hole punching)
- `net/netcheck` — NAT type detection, reflexive candidate discovery
- `derp` + `derp/derphttp` — relay server + client

**Build (ours):**
- `apps/derp` (new) — thin wrapper around `tailscale.com/derp` server: our TLS
  (step-ca), our region map, our metrics. Multi-region deploy.
- `apps/mesh-signal` (new) OR ride existing `agent-gateway` WSS — minimal
  coordination: peer endpoint exchange, pubkey distribution, DERP region map
  push. Speaks a small control surface to the embedded engine (not full
  tailscale control protocol — only what magicsock needs: peer set + endpoints
  + DERP map).
- `apps/agent/internal/mesh/engine.go` (new) — wire embedded `wgengine`+
  `magicsock`, feed it peer config from our registry, expose `octa0`-equivalent
  TUN.
- Identity bridge: step-ca leaf cert → node identity → mesh pubkey binding.
  Keep our enrollment; no tailscale auth-keys.

## Component breakdown & sub-effort

| Component | Build/Embed | Effort | Notes |
|-----------|-------------|--------|-------|
| userspace engine in agent | embed wgengine | M | replaces kernel wg-quick path |
| magicsock multiplexer | embed | S | used as-is |
| NAT detection (netcheck) | embed | S | needs STUN endpoints (run our own or public) |
| hole-punch (disco) | embed | S | used as-is |
| DERP server `apps/derp` | build (wrap) | M | TLS via step-ca, region map, HA |
| coordination/signaling | build | M | endpoint exchange + DERP map push |
| registry → engine feed | build | M | MeshPeer set → magicsock netmap |
| identity bridge step-ca↔pubkey | build | S | reuse enroll.go |
| STUN endpoints | run/embed | S | self-host or use public STUN |
| metrics/observability | build | S | handshake/path/relay stats |

Total ≈ **L-XL**, but most of the hard NAT/MTU code is embedded, not written.

## Risks

1. **Control-protocol impedance:** tailscale's engine expects a "netmap" shape.
   We must produce a minimal netmap from our registry without adopting the full
   tailscale control protocol. Scope this spike first.
2. **Userspace WG perf:** wireguard-go slower than kernel WG. Acceptable for
   most hosting traffic; benchmark for DB replication paths. Option: kernel WG
   for public-reachable pairs, userspace only where NAT traversal needed
   (hybrid) — adds complexity; defer.
3. **DERP HA:** relay is fallback path; multi-region + health-checked. Stateless
   per tailscale design — horizontal scale is fine.
4. **Vendoring drift:** pin tailscale module version; isolate behind our
   `internal/mesh` package so upgrades are contained.
5. **MTU:** userspace WG overhead ~80B; set TUN MTU accordingly; revisit for
   Phase 6 k8s double-encap.

## De-risking spike (do first, ~1 week)

Before committing Phase 4:
1. Vendor tailscale, stand up two userspace-WG nodes behind NAT (containers
   with simulated symmetric NAT).
2. Feed a hand-built netmap (2 peers + 1 DERP).
3. Prove: direct hole-punch succeeds; kill direct path → DERP fallback;
   restore → re-upgrade to direct.
4. Output: confirmed minimal netmap shape + the exact tailscale API surface we
   depend on. This locks the `mesh-signal` contract.

## Phase 0a revision (supersedes prior kernel-WG tasks)

- **T0a-1 (was `wireguard.configure` kernel action):** redefined → start the
  in-agent userspace mesh engine (`apps/agent/internal/mesh/engine.go`), no
  kernel `wg-quick`. Kernel-WG action kept only for public relay nodes if needed.
- **T0a-2 (node-side keygen):** unchanged and still critical — engine generates
  keypair locally, reports pubkey only.
- **T0a-3 (Hello mesh fields):** unchanged; add `derp_region` and reflexive
  candidates to reported endpoints.

## Sequencing within Phase 4

1. Spike (above) — gate.
2. `apps/derp` server (wrap tailscale derp + step-ca TLS).
3. Agent engine integration (embed wgengine/magicsock, netmap feed).
4. `mesh-signal` coordination (endpoint exchange + DERP map).
5. Path-migration + fallback validation in real NAT scenarios.
6. Metrics + ops surface.
