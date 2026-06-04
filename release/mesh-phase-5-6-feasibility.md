# Mesh Overlay — Phase 5 (Subnet Routing) + Phase 6 (k8s Federation) Feasibility

**Status:** Approved direction
**Date:** 2026-06-04
**Context:** Strategy A (embedded tailscale engine) makes both phases cheaper —
subnet routing is native in tailscale; k8s federation rides the mesh's flat L3.

---

## Phase 5 — Subnet Routing / Exit Node

### Key fact: tailscale already does this
With Strategy A (embedded `wgengine`/`magicsock` + `tailcfg` netmap), subnet
routing is a **native concept**, not new datapath code:
- A peer advertises routes (CIDRs) in its netmap entry.
- Other peers receive those routes; magicsock programs them → traffic to that
  CIDR flows to the advertising peer.
- Exit node = advertise `0.0.0.0/0` (+ `::/0`), opt-in per peer.

So Phase 5 is mostly **control-plane plumbing + approval**, not engine work.

### Tasks
- **P5-1 · Advertise field:** `MeshPeer.advertised_subnets` (already in Phase 0b
  schema) → emit into the peer's netmap entry (`mesh-signal` netmap builder).
- **P5-2 · Route approval gate:** advertised subnets are **not** auto-trusted.
  Operator/ACL approves which advertised CIDRs are accepted by which peers
  (tailscale model: routes advertised ≠ routes enabled). Add
  `mesh_route_approvals` (peer → approved CIDR). Filament toggle.
- **P5-3 · ACL integration:** approved subnet routes feed the Phase 2 AllowedIPs
  compiler — a route is only installed on peers whose ACL permits the
  destination group.
- **P5-4 · Exit node:** `MeshPeer.is_exit_node` + opt-in client setting; advertise
  default route; ACL-gated.
- **Effort:** **S-M** (native engine support; work is approval UX + ACL wiring).
- **Output:** reach private VPC/on-prem subnets via a gateway peer; pod-CIDR
  advertisement primitive ready for Phase 6.

### Risk
- Route conflicts / overlapping advertised CIDRs across peers — approval gate
  must detect + warn. (k8s overlap handled separately by Submariner Globalnet.)

---

## Phase 6 — k8s Federation (Submariner + our mesh)

### The integration insight
Our mesh (Phase 4 + 5) makes **every cluster gateway node appear on one flat,
mutually-routable L3 network** — across clouds and NAT. Submariner has a
**`vxlan` cable driver** designed exactly for "gateways already share an L3
network" (on-prem flat-network case). So we do NOT need to write a custom
Submariner cable driver: the mesh provides the flat L3, Submariner runs its
`vxlan` driver on top.

Two integration models — pick during a Phase 6 spike:

**Model 1 (preferred) — Submariner vxlan over mesh:**
- Each cluster gateway node = our mesh peer (runs agent + userspace engine).
- Submariner deployed with `--cable-driver vxlan`; gateways reach each other via
  the mesh's flat L3.
- Submariner owns the k8s layer: Lighthouse (cross-cluster DNS),
  Globalnet (CIDR overlap SNAT), ServiceExport/Import CRDs, route-agent.
- We own: mesh transport + gateway connectivity + identity.

**Model 2 (lighter, riskier) — mesh routes + Submariner control only:**
- Mesh advertises each cluster's pod/svc CIDR as subnet routes (Phase 5);
  mesh IS the inter-cluster datapath.
- Submariner used ONLY for Lighthouse + Globalnet + ServiceExport/Import; its
  cable/datapath disabled or no-op.
- Less moving parts in the datapath, but fighting Submariner's assumption that
  it owns the tunnel — more integration friction. Evaluate in spike.

### Tasks
- **P6-1 · Node type + edge:** `k8s-cluster` node kind →
  `SpecLoader.php` VALID_KINDS, `dispatcher.go` `urlForKind`; edge type
  `service-link`. **No CNI requirement.**
- **P6-2 · `apps/k8s-controller` (new Go):** orchestrate per-cluster:
  - Deploy Submariner broker (one cluster or external) + `submariner-operator`
    per cluster via `subctl join` using kubeconfig from Vault
    (`Cluster.kubeconfig_vault_ref`).
  - Configure cable driver = vxlan (Model 1).
  - Ensure gateway node is enrolled as a mesh peer + advertises pod/svc CIDR
    (ties to Phase 5 P5-1).
- **P6-3 · `__k8s-federation__` aggregate pass:** mirror the `__mesh__` pattern
  in `dispatcher.go` (~line 89) — whole-topology federation reconcile across all
  `k8s-cluster` nodes.
- **P6-4 · Globalnet auto-enable:** detect overlapping pod/svc CIDRs across
  clusters → enable Submariner Globalnet automatically (removes the
  non-overlapping-CIDR product constraint).
- **P6-5 · ReconcileSink handler:** ingest `__k8s-federation__` outputs in
  `app/Services/Topology/ReconcileSink.php` (kubeconfig refs → Vault, status →
  topology_nodes); existing `__mesh__` handler is the template.
- **P6-6 · TopologyDeployer path:** `TopologyDeployer.php` already dispatches k8s
  (`enqueueKubectlApply`) — add the federation join path.
- **Effort:** **L** (controller + Submariner orchestration; datapath reused).
- **Output:** provider k8s ↔ custom k8s cross-cluster services, NAT/VPC/CNI
  independent.

### De-risking spike (before committing Phase 6, ~1 week)
1. Two kind/k3s clusters, each with a gateway node on our mesh (use Phase 4
   spike output for mesh connectivity).
2. Deploy Submariner vxlan cable driver across them over the mesh.
3. `ServiceExport` a service in cluster A; resolve + reach it from cluster B via
   `*.clusterset.local` (Lighthouse).
4. Test overlapping pod CIDRs → Globalnet.
5. **Gate:** cross-cluster service reachable + DNS resolves + overlap handled.
   If Submariner vxlan-over-mesh fights the mesh, fall back to Model 2 or a
   custom cable driver.

### Risks
1. **vxlan-over-WireGuard double encap → MTU:** mesh TUN MTU minus vxlan
   overhead; set pod MTU accordingly. Measure in spike.
2. **Submariner broker placement:** SPOF if single; HA or external broker.
3. **kube-proxy mode / CNI quirks:** Submariner is CNI-agnostic but interacts
   with kube-proxy; validate against Calico + Flannel + Cilium in spike.
4. **Version coupling:** pin Submariner version; isolate behind `k8s-controller`.

---

## Combined effort

| Phase | Effort | New service | Notes |
|-------|--------|-------------|-------|
| 5 Subnet/exit | S-M | — | native in tailscale engine; UX+ACL work |
| 6 k8s federation | L | `apps/k8s-controller` (+ Submariner) | datapath reused from mesh; spike-gated |

Both depend on Phase 4 (mesh fabric). Phase 5 also feeds Phase 6 (pod-CIDR
advertisement). Recommend the Phase 6 spike reuse the Phase 4 spike lab.
