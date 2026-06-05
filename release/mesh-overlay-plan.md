# Podmaker Mesh Overlay + k8s Federation Plan

> **Durable design extracted** to [`docs/reference/mesh-overlay-architecture.md`](../reference/mesh-overlay-architecture.md).
> This document is retained for its roadmap cross-links (`mesh-roadmap.md`,
> `mesh-phase-*-tasks.md`); consult the reference for the canonical architecture.

**Status:** Feature-complete in code across all layers (6 phases + agent daemon
+ operator UI + 3 new services), unit-tested at every hop. Remaining is
deploy-time / external-infra only (real clusters, CA, node TUN, gateway). See
`mesh-roadmap.md` § Implementation status.
**Date:** 2026-06-04

## Goal

Unite provider-provisioned nodes (AWS/GCP/Hetzner/DO), customer-attached
custom VPS, and Kubernetes clusters into a single Zero-Trust WireGuard mesh.
One flat private network, one name space, one access policy — independent of
physical location, NAT, cloud provider, or CNI.

This is the differentiator vs single-host panels (Panelica) and pure cloud
panels: hybrid "bring your own server / cluster + our cloud" works as one
fabric. Inspiration taken from OctaMesh / Tailscale (mesh) and Submariner
(k8s federation) — but transport stays ours.

## Locked decisions

1. **Data plane:** full-mesh P2P with STUN/ICE hole-punching, DERP relay
   fallback. (Not hub-and-spoke — though `mesh_hub.go` may seed an interim
   reachable-peer mode for MVP-1.)
2. **k8s federation:** CNI-agnostic. **Cilium ClusterMesh rejected** (CNI lock
   narrows market). Use **Submariner with our mesh as the cable-driver**
   (option B). Submariner provides Lighthouse DNS + Globalnet CIDR-conflict
   handling + ServiceExport/Import; our WireGuard mesh is the transport.
3. **CNI requirement:** none. Works with any CNI (Calico/Flannel/Cilium/...).

## Current baseline (≈35-40% ready)

- `apps/mesh-controller` — `WGPeer`/`WGMesh`, deterministic addressing,
  `Render()`, `mesh_hub.go` hub topology skeleton + CoreDNS zone stubs.
  `__mesh__` whole-topology reconcile aggregate.
  - Gap: ACL, DERP, MagicDNS, subnet routing. AllowedIPs hardcoded `/32`.
  - Keypair generated controller-side (`wireguard.go:136`) — must move to node.
- `apps/agent` — step-ca enroll + HMAC frame protocol (`Hello`/`Heart`),
  gopsutil metrics. **No WireGuard interface management.**
- `apps/topology-planner` — `NodeSpec`/`EdgeSpec`, `urlForKind` dispatch,
  `__mesh__` aggregate pattern. k8s partial (`Cluster` model, `kubernetes`
  config field, `enqueueKubectlApply`).
- `apps/license-api/internal/ce` — Ed25519-signed heartbeat, suitable to carry
  posture attestation.

## Phases

### Phase 0 — Foundation (prereq for everything) · L
- **0a. Agent WireGuard data-plane (userspace):** REVISED per Phase 4 decision —
  use **userspace WireGuard** (`wireguard-go` + tailscale `magicsock`), NOT
  kernel `wg-quick`. Kernel WG cannot do DERP failover (fixed per-peer
  Endpoint). New `apps/agent/internal/mesh/engine.go` starts the in-agent
  engine. Extend `Hello` frame with `wireguard_pub_key`, `mesh_address`,
  `mesh_endpoints[]` (`apps/agent/internal/session/messages.go`).
  **Keypair generated on the node** (private key never leaves host) — required
  for the Zero-Trust claim. Kernel `wg-quick` action retained only for
  always-public relay nodes. See `mesh-phase-4-feasibility.md`.
- **0b. Mesh peer registry (control-plane):** new `MeshPeer` model + migration
  (`node_ref`, `public_key`, `mesh_ip`, `endpoints[]`, `posture_state`,
  `admission_state`). mesh-controller becomes stateless render; control-plane
  owns state.
- **0c. Reconcile contract:** add to `ReconcileRequest.Config`: `acl_policies`,
  `derp_relays`, `dns_mode=magicdns`, `subnet_routes`, `peer_endpoints`
  (`packages/shared-go/pkg/controller/types.go`). Add to `WGPeer`:
  `AllowedEndpoints[]`, `AdvertisedSubnets[]`, `Persistent`
  (`packages/shared-go/pkg/playbook/wireguard.go`).

### Phase 1 — MagicDNS · S · most visible
- `dns_mode=magicdns` in mesh-controller: per-peer CoreDNS zone (`node.mesh` →
  mesh_ip); finish `CoreDNSZone/Corefile` (`mesh_hub.go:133`).
- Agent: CoreDNS stub / resolv.conf integration. Roaming IP → registry change →
  zone regenerate.
- Depends: 0b. Output: `db-primary.mesh` resolves from every node.

### Phase 2 — Mesh ACL + SSH access · M
- ACL compiler `compileAllowedIPs(policies, peer, allPeers)` in mesh-controller.
  Add `peer-group` concept (`TopologyNode.config.mesh_group`).
- Policy `from_group → to_group : ports`. L3 via AllowedIPs kernel routes +
  L4 port restriction via nftables on node (WireGuard alone can't filter ports).
- Mesh SSH: extend step-ca (`enroll.go`) to SSH cert authority; SSO identity →
  principal.
- Depends: 0. Output: `app-tier → db-tier:5432 only`, default-deny.

### Phase 3 — Posture checks · M
- Add `PostureAttestation` to heartbeat envelope (OS patch, disk-encryption/LUKS,
  EDR, firewall) — `apps/license-api/internal/ce/heartbeat.go:31`. Ed25519
  signing already present.
- Agent collects: extend gopsutil collector + `cryptsetup`/`/etc/os-release`
  parsing.
- Admission gate: posture audit table; `mesh_admission_state != approved` →
  registry removes peer from mesh (no route injection). Continuous re-eval each
  heartbeat.
- Depends: 0b. Output: unhealthy VPS can't join mesh.

### Phase 4 — NAT traversal / DERP + P2P signaling · XL · critical path
**Strategy A (locked):** embed tailscale data-plane libs (`wgengine`,
`magicsock`, `disco`, `net/netcheck`, `derp`); control/identity/UI stay ours.
Full detail + de-risking spike in `mesh-phase-4-feasibility.md`.
- **Embedded (vendor tailscale, BSD-3):** userspace WG engine, magicsock
  multiplexer, disco hole-punch, netcheck NAT detection, DERP client/server.
- **`apps/derp`** (new, build): thin wrapper over tailscale `derp` server with
  step-ca TLS + our region map. Multi-region, stateless, HA.
- **`apps/mesh-signal`** (new, build — or ride agent-gateway WSS): minimal
  coordination — peer endpoint exchange, pubkey distribution, DERP map push.
  Produces a minimal "netmap" for magicsock from our MeshPeer registry (NOT the
  full tailscale control protocol).
- **Identity:** step-ca leaf → node identity → mesh pubkey. No tailscale
  auth-keys.
- **Do first:** ~1-week spike proving direct hole-punch + DERP fallback +
  re-upgrade between two simulated-symmetric-NAT nodes; locks the netmap shape.
- Depends: 0a (userspace engine). Output: custom VPS behind NAT/symmetric-NAT
  joins the mesh.
- **Highest risk.** Control-protocol impedance, userspace-WG perf, DERP HA, MTU.

### Phase 5 — Subnet routing / exit node · M · prereq for k8s
- `WGPeer.AdvertisedSubnets` → gateway peer advertises CIDR, added to other
  peers' AllowedIPs (same compiler seam as Phase 2).
- Exit node: peer advertises `0.0.0.0/0`, opt-in.
- Depends: 2. Output: reach private VPC/on-prem subnet via gateway — and the
  foundation for k8s pod-CIDR routing.

### Phase 6 — k8s federation (Submariner + mesh cable-driver / Arch B) · L
- New node type `k8s-cluster` → `SpecLoader.php` VALID_KINDS, `dispatcher.go`
  `urlForKind`, edge type `service-link`. **No CNI requirement.**
- New `apps/k8s-controller` (Go): deploy Submariner broker + submariner-operator
  per cluster; orchestrate `subctl join` using kubeconfig from Vault
  (`Cluster.kubeconfig_vault_ref`). `__k8s-federation__` aggregate pass (copy the
  `__mesh__` pattern, `dispatcher.go:89`).
- **mesh-cable-driver adapter:** Submariner gateway → our WireGuard mesh peer.
  ClusterMesh tunnel rides the mesh overlay (Phase 4 + 5).
- Globalnet auto-enabled for overlapping CIDRs (no non-overlap product rule
  needed).
- `TopologyDeployer.php:67` already dispatches k8s — add the federation path.
- Depends: 4 + 5. Output: provider k8s ↔ custom k8s cross-cluster service,
  NAT/VPC/CNI independent.

## Sequencing

```
Phase 0 (foundation) ──┬─► 1 MagicDNS          [fast win, demo]
                       ├─► 2 ACL ──► 5 Subnet ──► 6 k8s
                       ├─► 3 Posture
                       └─► 4 NAT/DERP ──────────► 6 k8s
```

- **MVP-1 (unification demo):** 0 + 1, direct-reachable peers → "resolve by name,
  one mesh". No NAT yet.
- **MVP-2 (true hybrid):** + 4 → NAT'd custom VPS actually joins. Main selling
  point.
- **Enterprise:** + 2/3 (SOC2 story).
- **k8s:** + 5/6.

## Effort & new services

| Phase | Effort | New service |
|-------|--------|-------------|
| 0 Foundation | L | — (registry + agent WG) |
| 1 MagicDNS | S | — |
| 2 ACL/SSH | M | — |
| 3 Posture | M | — |
| 4 NAT/DERP | XL | `mesh-signal`, `derp` |
| 5 Subnet | M | — |
| 6 k8s federation | L | `k8s-controller` (+ Submariner) |

≈ 2 new Go services (signal, DERP) + 1 controller (k8s). Existing
`mesh-controller`/`agent`/`topology-planner` extend deeply.

## Top risks

1. **Phase 4 (DERP/hole-punch)** — critical path, most uncertain. Evaluate
   tailscale `tsnet`/DERP fork before writing from scratch.
2. **Private key egress** — today mesh-controller generates keys; move to
   node-side generation or the Zero-Trust claim is hollow.
3. ~~k8s CNI lock~~ — **resolved** by choosing CNI-agnostic Submariner+mesh (B).
4. **MTU** — WireGuard + CNI double-encap; lower pod MTU.
