# Podmaker Mesh Overlay + k8s Federation — Master Roadmap

**Status:** Design approved; implementation not started
**Date:** 2026-06-04
**Owner doc set:** this is the index. Detailed docs linked per phase.

## One-line goal
Turn the existing `mesh-controller`/`agent`/`topology-planner` into a
Zero-Trust WireGuard overlay that unites provider-provisioned nodes +
customer-attached custom VPS + Kubernetes clusters into one flat private
network — independent of location, NAT, cloud provider, or CNI.

## Why (positioning)
- Real benchmark = **Panelica** (single-host panel); we sit a layer up
  (orchestration / multi-cloud). The mesh is the differentiator neither
  single-host panels nor pure cloud panels can match: hybrid "bring your own
  server / cluster + our cloud" as one fabric.
- Inspiration: **OctaMesh / Tailscale** (mesh data plane), **Submariner**
  (k8s federation). Transport stays ours; we don't compete in ZTNA — we embed it.

## Locked decisions (decision registry)
| # | Decision | Rationale |
|---|----------|-----------|
| D1 | Full-mesh **P2P + DERP fallback** (not hub-and-spoke) | latency/scale; OctaMesh-grade |
| D2 | Phase 4 = **Strategy A: embed tailscale data-plane libs** | reuse battle-tested NAT/DERP; transport isn't our differentiator |
| D3 | **Userspace WireGuard** (wireguard-go), not kernel wg-quick | kernel WG can't DERP-failover (fixed Endpoint) |
| D4 | Identity stays **step-ca**; no tailscale auth-keys / no headscale | single fabric, single identity |
| D5 | k8s federation **CNI-agnostic**; Cilium ClusterMesh **rejected** | CNI lock narrows market |
| D6 | k8s = **Submariner + our mesh** (vxlan cable driver over mesh flat L3) | Lighthouse DNS + Globalnet + ServiceExport/Import free; no custom cable driver |
| D7 | **Node-side keygen** (private key never leaves host) | Zero-Trust claim integrity |

## Phase map

| Phase | What | Effort | New services | Detailed doc |
|-------|------|--------|--------------|--------------|
| 0 | Foundation: userspace engine, peer registry, reconcile contract | L | — | `mesh-phase-0-1-tasks.md` |
| 1 | MagicDNS (`name.mesh`) | S | — | **DONE (2026-06-04)** — `mesh-phase-0-1-tasks.md` |
| 2 | Mesh ACL (L4 FilterRule) + SSH (step-ca SSH CA) | M | — | **ACL core + SSH skeleton done (2026-06-04)** — `mesh-phase-2-3-tasks.md` |
| 3 | Posture checks (heartbeat attestation + admission gate) | M | — | **core done (2026-06-04)** — `mesh-phase-2-3-tasks.md` |
| 4 | NAT traversal / DERP + P2P signaling | XL | `mesh-signal`, `derp` | `mesh-phase-4-feasibility.md`, `mesh-phase-4-spike-tasks.md` |
| 5 | Subnet routing / exit node | S-M | — | **core done (2026-06-04)** — `mesh-phase-5-6-feasibility.md` |
| 6 | k8s federation (Submariner + mesh) | L | `k8s-controller` | **skeleton done (2026-06-04)** — `mesh-phase-5-6-feasibility.md` |

Plan overview: `mesh-overlay-plan.md`.

## Dependency graph
```
                       ┌─► 1 MagicDNS ──────────────────────────► MVP-1
                       │
0 Foundation ──────────┼─► 3 Posture (independent) ─────────────► Enterprise
(userspace engine,     │
 registry, contract)   ├─► 4 NAT/DERP ──┬─► (MVP-2: NAT'd VPS) ──► MVP-2
                       │   [spike-gated] │
                       │                 └─► 5 Subnet ─► 6 k8s ──► k8s GA
                       │                                  [spike-gated]
                       └─► 2 ACL/SSH ────────────────────┘
                           (needs netmap shape from 4 spike)
```
Notes:
- **0 blocks everything.**
- **2 ACL** depends on the netmap shape locked by the **Phase 4 spike** (shared
  netmap-builder seam) — start 2 after that spike.
- **3 Posture** is independent of 4 — can run parallel once 0b registry exists.
- **5** depends on 4 (mesh fabric); **6** depends on 5 (pod-CIDR advertise) + 4.
- Two spike gates: Phase 4 spike (`mesh-phase-4-spike-tasks.md`) and a Phase 6
  Submariner-over-mesh spike (reuses the Phase 4 lab).

## Milestones
| Milestone | Phases | Demo-able outcome |
|-----------|--------|-------------------|
| **MVP-1** | 0 + 1 | Two direct-reachable nodes form a mesh; resolve each other by `name.mesh`. No NAT yet. |
| **MVP-2** | + 4 | Custom VPS behind NAT/symmetric-NAT actually joins; P2P with DERP fallback. **Main selling point.** |
| **Enterprise** | + 2 + 3 | Default-deny ACL, L4 micro-segmentation, posture-gated admission, SSO SSH. SOC2 story. |
| **k8s GA** | + 5 + 6 | Provider k8s ↔ custom k8s cross-cluster services, NAT/VPC/CNI independent. |

## Critical path & risks
- **Critical path:** 0 → 4 (spike → derp → engine → signal) → 5 → 6. Phase 4 is
  the long pole (XL) and the riskiest.
- **Do first:** Phase 4 ~1-week de-risking spike. If it fails, reconsider D2/D4
  (headscale) before sinking Phase 4 cost.
- **Top risks:** (1) tailscale control-protocol impedance / netmap shape;
  (2) userspace-WG perf on DB-replication paths; (3) private-key egress —
  enforce node-side keygen (D7); (4) MTU under WG (+ vxlan double-encap in
  Phase 6); (5) Submariner version/CNI coupling.

## Baseline readiness
≈35-40%. Existing: WireGuard render skeleton + hub mesh + CoreDNS stubs
(`mesh-controller`), step-ca enroll + HMAC frame protocol (`agent`),
`__mesh__` aggregate dispatch (`topology-planner`), Ed25519 heartbeat
(`license-api/ce`). Missing: userspace engine, DERP, signaling, registry as
source-of-truth, MagicDNS resolver, ACL compiler, posture, k8s-controller.

## Net-new services summary
- `apps/mesh-signal` — coordination: endpoint exchange, pubkey distribution,
  DERP map, builds minimal netmap from registry.
- `apps/derp` — relay (wraps tailscale derp + step-ca TLS), multi-region, HA.
- `apps/k8s-controller` — Submariner orchestration, federation reconcile.

## Implementation status (2026-06-04)

**Feature-complete in code across all layers** — datapath + control-plane +
transport + agent daemon + operator UI + services. What remains is deploy-time /
external-infra only (real clusters, real CA, real node TUN, real gateway).

| Phase | Built | Tests |
|-------|-------|-------|
| 0 Foundation | userspace engine (`apps/mesh-engine`, off-go.work), MeshPeer/MeshNetwork registry + IP allocator, reconcile contract, node-side keygen, stateless mesh-controller, planner wiring, Hello mesh fields, endpoint ingest | keygen ×4 |
| 1 MagicDNS | engine DNS resolver (`name.<domain>`→ip from netmap), agent split-DNS (resolvectl), Filament MeshPeers page | resolver ×2 |
| 2 ACL + SSH | engine L4 packet filter (tailcfg.FilterRule→tsTun), MeshAclPolicy + groups + compiler, MeshAclPolicyResource CRUD; engine sshd-config + principals render from netmap, principalFromEmail, principal UI | ACL ×2, ssh ×2 (engine) +2 (agent) |
| 3 Posture | agent CollectPosture, PostureEvaluator (default-lenient/fail-closed), admission gate in MeshPeerSync | (lint) |
| 4 Prod | spike proven (S0-S5), `apps/derp` (persistent key, step-ca TLS), mesh-signal = MeshNetmapBuilder + endpoint, gateway Poller (sign+dispatch), agent EngineClient + Supervisor + ApplyNetmapJSON | wire ×1, supervisor ×1, netmap ×3, derp smoke, poller ×2 |
| 5 Subnet/exit | approved_subnets gate (advertised≠enabled), MeshRouteApprover, route approval UI; exit node = approve 0.0.0.0/0 | (logic verified) |
| 6 k8s | `apps/k8s-controller` (Submariner vxlan-over-mesh orchestration), k8s-cluster kind + service-link edge, `__k8s-federation__` pass, ReconcileSink + TopologyDeployer wiring | reconcile ×5 |
| — Agent daemon | WSS transport (`transport.Dial`/`WSConn`/`DeriveSession`), session.Receiver (HMAC+replay), `run.Loop` (Hello+heartbeat+command+mesh routing), `playbook.OSRunner` (production exec runner), dispatcher wired; agent `run` went from stub → fully functional | receiver, loop ×1, osrunner ×2 |

**New services:** `apps/mesh-engine`, `apps/derp` (both off-go.work, embed tailscale), `apps/k8s-controller` (in go.work). Plus control-plane `app/Services/Mesh/*` + `app/Filament/Admin/{Pages/Mesh,Resources/MeshAclPolicyResource}`.

**Remaining — deploy-time / external-infra ONLY (no code):**
1. Real `subctl`/broker + step-ca SSH CA cert-mint runners against real clusters/CA.
2. Real-node (Linux+root TUN) + real-cluster + real-gateway integration tests.
3. Playbook signing pubkey + step-ca SSH CA provisioning at enroll/deploy time;
   `mesh.ssh_ca_pubkey_path` config.

The MVP-2 e2e chain (CP registry → builder → gateway poller → signed mesh_netmap
frame → agent WSS receive loop → supervisor → mesh-engine octa0) is code-complete
and unit-tested at every hop; it needs a real gateway + root TUN host to run live.

## Doc index
- `mesh-overlay-plan.md` — full phased plan + seams.
- `mesh-phase-0-1-tasks.md` — Phase 0+1 task breakdown.
- `mesh-phase-2-3-tasks.md` — Phase 2+3 task breakdown.
- `mesh-phase-4-feasibility.md` — Phase 4 strategy + kernel/userspace decision.
- `mesh-phase-4-spike-tasks.md` — Phase 4 de-risking spike.
- `mesh-phase-5-6-feasibility.md` — subnet routing + k8s federation.
- `mesh-roadmap.md` — this index.
