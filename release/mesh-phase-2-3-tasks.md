# Mesh Overlay — Phase 2 (ACL + SSH) + Phase 3 (Posture) Task Breakdown

**Status:** Approved direction
**Date:** 2026-06-04
**Context:** Strategy A (embedded tailscale engine) gives native L3+L4 packet
filtering — ACLs compile to `tailcfg.FilterRule` enforced in the engine, no
nftables needed. Posture reuses the CE Ed25519 heartbeat.

---

## Phase 2 — Mesh ACL + SSH access

### Key fact: tailscale engine enforces L4 filters natively
The embedded engine compiles per-node packet filters (src CIDR → dst CIDR :
proto/ports). So "app-tier → db-tier:5432 only, default-deny" is expressed as
filter rules in each peer's netmap entry and enforced in magicsock — **not** via
node nftables. This is cheaper and tamper-resistant (enforced before the packet
reaches the host stack).

### T2-1 · Peer-group model
- **Files (new):** `apps/control-plane/app/Models/MeshGroup.php` +
  migration; `mesh_peer_group` pivot (peer ↔ group). Or simpler:
  `MeshPeer.groups` jsonb tag list (tailscale `tag:` style).
- Groups assignable from topology (`TopologyNode.config.mesh_group`) and to
  attached VPS / clusters.
- **Accept:** a peer resolves to ≥0 groups; groups usable as policy src/dst.

### T2-2 · ACL policy model + DSL
- **Files (new):** `apps/control-plane/app/Models/MeshAclPolicy.php` + migration.
- Rule shape: `{action: accept, src: [group|cidr|*], dst: [group|cidr|*],
  proto: tcp|udp|*, ports: [int|range]}`. Default-deny baseline.
- Optional: store as a single HuJSON document per MeshNetwork (tailscale ACL
  format) for familiarity + import/export.
- **Accept:** policy CRUD; validation rejects unknown group refs.

### T2-3 · ACL → FilterRule compiler
- **File (new):** `apps/control-plane/app/Services/Mesh/AclCompiler.php` (or in
  `mesh-signal` Go if netmap is built there).
- Resolve groups → member mesh_ips; emit per-peer `tailcfg.FilterRule` set into
  the netmap (`mesh-signal` netmap builder). Recompile on policy/peer change.
- **Accept:** `app→db:5432` permitted, `app→db:3306` denied, observed at engine
  packet-filter level on the node.

### T2-4 · Schedule + templates (OctaMesh parity, optional MVP+)
- Time-bound activation (`active_window`) on a policy → compiler emits/withdraws
  rules on schedule. Policy templates: "dev→staging", "admin→prod", "ci→deploy".
- **Accept:** policy auto-revokes when window closes.

### T2-5 · Mesh SSH (step-ca SSH CA)
- Extend step-ca usage (`apps/agent/internal/enroll`) to issue **SSH host +
  user certs** (step-ca supports SSH CA). SSO identity → SSH principal mapping.
- SSH reachability still gated by ACL (T2-3) on port 22 over mesh.
- **File (new):** `apps/control-plane/app/Services/Mesh/SshCaService.php` +
  agent-side `internal/mesh/ssh.go` (write `authorized_principals` / host cert).
- **Accept:** SSO-authenticated operator gets short-lived SSH cert; reaches only
  ACL-permitted peers; cert expiry forces re-auth.

### Effort: **M** (engine enforces filters natively; work is model + compiler + SSH CA).

---

## Phase 3 — Posture checks

### Reuse: CE heartbeat (Ed25519-signed) carries attestation
Heartbeat infra already signs canonical JSON with per-install Ed25519 keys
(`apps/license-api/internal/ce/heartbeat.go`). Extend it to carry posture; gate
mesh admission on the result.

### T3-1 · Posture attestation in heartbeat envelope
- **File:** `apps/license-api/internal/ce/heartbeat.go` — add optional
  `PostureAttestation` to `HeartbeatEnvelope`:
  `{os_version, os_patch_level, disk_encrypted, encryption_alg, edr_status,
  edr_product, firewall_enabled, selinux_status, attested_at}`.
- Mirror in control-plane envelope (`app/Services/Pairing/HeartbeatRunner.php`).
- **Accept:** signed attestation round-trips; signature still verifies.

### T3-2 · Agent posture collector
- **File (new):** `apps/agent/internal/posture/collector.go`.
- Collect: `/etc/os-release` (distro/version), kernel patch (`uname -r` + pkg
  manager last-update), disk encryption (`cryptsetup status` / `lsblk`
  crypto type / BitLocker on Windows), EDR presence (process/service probe:
  crowdstrike/sentinel/...), firewall state (ufw/firewalld/nftables).
- Reuse gopsutil where applicable (`internal/metrics/collector.go`).
- **Accept:** attestation struct populated on Linux (Debian/RHEL/Alpine);
  graceful unknowns.

### T3-3 · Posture requirements + evaluation
- **File (new):** `apps/license-api/internal/premium/posture.go` (or control-plane
  service). Per-workspace/tier `PostureRequirements`
  `{require_disk_encryption, require_edr, require_firewall,
  max_os_patch_age_days}`.
- Evaluate attestation → `admission_state` (admitted|denied|pending_remediation)
  + `violations[]`.
- **Accept:** missing disk encryption → denied with reason.

### T3-4 · Storage + audit
- **Migration:** extend `ce_installs` (or `MeshPeer`) with `last_posture`,
  `last_posture_at`, `posture_state`, `violations`. New `ce_posture_audits`
  table (forensics: attestation + verdict + timestamp).
- **Accept:** every attestation audited; history queryable per install.

### T3-5 · Mesh admission gate
- **File:** `app/Services/Mesh/MeshPeerSync.php` — on posture verdict, set
  `MeshPeer.admission_state`. If not `admitted`: remove peer from netmap (no
  routes/filters) → engine isolates it. Continuous re-eval each heartbeat;
  failing peer isolated immediately.
- **Accept:** a peer that fails posture mid-session is dropped from the mesh
  within one heartbeat interval; recovers when posture passes.

### T3-6 · MDM/EDR integration (OctaMesh parity, optional MVP+)
- Connectors: Jamf / Intune / CrowdStrike / SentinelOne API → enrich/override
  attestation with authoritative device health.
- **Accept:** EDR API "non-compliant" → denied regardless of self-report.

### Effort: **M**.

---

## Combined effort + sequencing

| Phase | Effort | New services | Depends |
|-------|--------|--------------|---------|
| 2 ACL/SSH | M | — | 0 (+ netmap builder from 4) |
| 3 Posture | M | — | 0b registry, CE heartbeat |

- Phase 2 ACL compiler shares the netmap-builder seam with Phase 4/5 — sequence
  after the Phase 4 spike locks the netmap shape.
- Phase 3 is independent of Phase 4 — can proceed in parallel once Phase 0b
  registry exists.
- Both are the **enterprise/SOC2 story**; not on the MVP-1/MVP-2 critical path.

## Note on enforcement layers
- L3 reachability: netmap AllowedIPs (Phase 0c/2).
- L4 port filter: tailscale FilterRule in engine (Phase 2) — preferred.
- Defense-in-depth (optional): node nftables mirror of the same policy.
