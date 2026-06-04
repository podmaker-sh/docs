# Mesh Overlay — Phase 0 + 1 Task Breakdown

Implementation-ready breakdown for Phase 0 (Foundation) and Phase 1 (MagicDNS)
of `mesh-overlay-plan.md`. Task IDs are stable references for commits/PRs.

Verify exact line numbers against current code before editing — structs/paths
below are the reliable anchors.

---

## Phase 0a — Agent WireGuard data-plane

### T0a-1 · In-agent userspace mesh engine  ⚠️ REVISED (Phase 4 decision)
- **Was:** kernel `wg-quick` playbook action. **Now:** userspace WireGuard
  engine (kernel WG can't do DERP failover — fixed per-peer Endpoint). See
  `mesh-phase-4-feasibility.md`.
- **File (new):** `apps/agent/internal/mesh/engine.go` — embed tailscale
  `wgengine` + `magicsock` (vendored, BSD-3). Bring up TUN, feed peer set from
  registry-sourced netmap, expose handshake/path status.
- **Phase-0/MVP-1 scope:** engine with **direct endpoints only** (no DERP yet —
  DERP lands in Phase 4). Kernel `wg-quick` action retained ONLY for
  always-public relay nodes if needed.
- **Params (from reconcile):** `interface` (`octa0`), `address` (`10.42.x.y/32`),
  `listen_port`, `peers[]` (`public_key`, `allowed_ips`, `endpoint?`), `mtu?`.
- **Behavior:** idempotent reconcile of peer set into the running engine (live
  add/remove without dropping existing handshakes).
- **Accept:** two direct-reachable userspace-WG nodes handshake; peer add/remove
  applied live; no kernel `wg-quick` dependency.

### T0a-2 · Node-side key generation (security-critical)
- **Move keypair gen from controller to agent.** Today
  `packages/shared-go/pkg/playbook/wireguard.go` `generateKeyPair()` runs
  controller-side and private key ships in reconcile output. **Stop shipping
  private keys.**
- Agent generates Curve25519 keypair on first mesh-join, persists private key
  `0600` at `/etc/podmaker/agent/wg_private.key`, reports **public key only**.
- **Files:** new `apps/agent/internal/mesh/keys.go`; remove private-key field
  from reconcile peer output (`apps/mesh-controller/cmd/mesh-controller/main.go`
  peer map — drop `private_key`).
- **Accept:** private key never present in any HTTP payload / Vault write /
  control-plane log. Grep CI check.

### T0a-3 · Extend agent `Hello` frame with mesh fields
- **File:** `apps/agent/internal/session/messages.go` (`Hello` struct) + mirror
  in `apps/agent-gateway/internal/session` + `registry.Agent`.
- **Add:** `mesh_capable bool`, `wireguard_pub_key string`, `mesh_address string`,
  `mesh_endpoints []string` (local + reflexive candidates).
- Gateway on `Hello`: persist to registry, forward to control-plane sink
  (`apps/agent-gateway/internal/sink`).
- **Accept:** control-plane receives pubkey + endpoints on agent connect.

---

## Phase 0b — Mesh peer registry (control-plane, source of truth)

### T0b-1 · `MeshPeer` model + migration
- **Files (new):** `apps/control-plane/app/Models/MeshPeer.php`,
  `apps/control-plane/database/migrations/XXXX_create_mesh_peers_table.php`.
- **Columns:** `id` (ulid), `workspace_id` fk, `mesh_network_id` fk,
  `node_ref_type` (server|attached_vps|cluster), `node_ref_id`,
  `public_key`, `mesh_ip` (unique per network), `endpoints` jsonb,
  `advertised_subnets` jsonb, `posture_state` (unknown|approved|denied),
  `admission_state` (pending|admitted|isolated), `last_handshake_at`,
  `last_seen_at`, timestamps.
- **Plus** `MeshNetwork` model + table: `workspace_id`, `name`, `cidr`
  (default `10.42.0.0/24`), `listen_port`, `dns_mode`, `dns_domain`
  (default `mesh`).
- **Accept:** one MeshPeer per mesh-capable node; deterministic mesh_ip
  allocation moves here (out of `wireguard.go:NewMesh`).

### T0b-2 · IP allocation service
- **File (new):** `apps/control-plane/app/Services/Mesh/MeshIpAllocator.php`.
- Allocate next free `/32` in network CIDR, persist on MeshPeer, idempotent by
  `node_ref`. Replaces lexicographic deterministic alloc in
  `shared-go/pkg/playbook/wireguard.go`.
- **Accept:** stable IP across reconciles; no collision; reclaim on detach.

### T0b-3 · Endpoint ingest from gateway
- **File (new):** `apps/control-plane/app/Services/Mesh/MeshPeerSync.php`,
  consumed by the agent-gateway sink endpoint that handles `Hello`/`Heartbeat`.
- On Hello: upsert MeshPeer (pubkey, endpoints). On Heartbeat: update
  `last_seen_at`, `last_handshake_at`.
- **Accept:** roaming endpoint change persisted within one heartbeat.

### T0b-4 · mesh-controller becomes stateless renderer
- **File:** `apps/mesh-controller/cmd/mesh-controller/main.go`.
- Stop generating keys/IPs. `ReconcileRequest.Config` now carries the full peer
  set (pubkeys, mesh_ips, endpoints, allowed_ips) from control-plane registry.
  Controller only renders `wg` config + DNS artifacts.
- **Accept:** controller restart produces identical output for same input;
  no in-memory state.

---

## Phase 0c — Reconcile contract extension

### T0c-1 · Extend shared types
- **File:** `packages/shared-go/pkg/controller/types.go`.
- `ReconcileRequest.Config` keys (documented constants): `acl_policies`,
  `derp_relays`, `dns_mode` (`hosts|coredns|magicdns`), `subnet_routes`,
  `peer_endpoints`, `peers` (full peer objects).
- **File:** `packages/shared-go/pkg/playbook/wireguard.go` — `WGPeer` add
  `AllowedEndpoints []string`, `AdvertisedSubnets []string`, `Persistent bool`.
  Keep `Render()` backward-compatible (new fields optional).
- **Accept:** existing topologies render unchanged when new keys absent.

### T0c-2 · Planner passes registry peers
- **File:** `apps/topology-planner/internal/apply/dispatcher.go` (the `__mesh__`
  pass, ~line 89). Source peer set from control-plane registry instead of
  generating endpoints inline.
- **Accept:** `__mesh__` reconcile carries `peers` array sourced from MeshPeer.

---

## Phase 1 — MagicDNS

### T1-1 · CoreDNS zone generation in mesh-controller
- **File:** `apps/mesh-controller/cmd/mesh-controller/main.go` +
  `packages/shared-go/pkg/playbook/mesh_hub.go` (finish `CoreDNSZone()` /
  `CoreDNSCorefile()` stubs).
- `dns_mode=magicdns`: emit zone `name.<domain>` → mesh_ip for every peer in
  network. Output `dns_zone` + `dns_corefile` in `ReconcileResponse.Outputs`.
- **Accept:** N peers → N A-records; deterministic, sorted.

### T1-2 · Agent DNS integration
- **File (new):** `apps/agent/internal/mesh/dns.go` + playbook action
  `coredns.configure` (or resolv.conf stub injector).
- Option A (preferred): run lightweight CoreDNS on each node bound to mesh_ip,
  `<domain>` zone local, forward everything else to system resolver.
- Option B (MVP): `/etc/hosts` injection (reuse existing
  `playbook/hosts.go RenderHostsScript`) — static, no roaming.
- Ship A; keep B as fallback flag.
- **Accept:** `dig db-primary.mesh` from any node resolves to its mesh_ip.

### T1-3 · Zone regenerate on topology change
- **File:** `apps/control-plane/app/Services/Mesh/MeshPeerSync.php` → on
  MeshPeer add/remove/IP-change, trigger mesh reconcile (re-render zone, push
  `coredns.configure` playbook via PendingCommand).
- **Accept:** new node joins → name resolvable on all peers within one reconcile.

### T1-4 · Control-plane mesh UI surface (minimal)
- **File (new):** Filament page under `app/Filament/Admin/Pages/Mesh/` listing
  MeshNetwork + peers (name, mesh_ip, endpoint, last_handshake, admission_state).
  Read-only for MVP.
- **Accept:** operator sees mesh state; matches `wg show` on nodes.

---

## Suggested commit/PR sequence

1. T0c-1 (contract) — unblocks everything, low risk.
2. T0b-1..T0b-2 (registry + IP alloc).
3. T0a-1..T0a-2 (agent WG + node keygen).
4. T0a-3 + T0b-3 (endpoint flow).
5. T0b-4 + T0c-2 (stateless controller + planner wiring).
6. T1-1..T1-4 (MagicDNS).

**MVP-1 gate:** after step 6, two direct-reachable nodes form a mesh and resolve
each other by `name.mesh`. NAT traversal (Phase 4) not yet required.
