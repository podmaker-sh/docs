# mesh-signal ↔ engine Contract (locked by Phase 4 spike)

**Status:** Locked by `spikes/mesh-p4` (S5 deliverable)
**Date:** 2026-06-04
**Purpose:** the exact, minimal surface `apps/mesh-signal` must produce from the
`MeshPeer` registry to drive the embedded tailscale engine — derived from a
working spike, not from reading tailscale internals. Production `apps/agent`
engine + `apps/mesh-signal` are specced from this doc.

## Spike evidence
`spikes/mesh-p4/node` — three passing tests (run `GOWORK=off go test ./node/ -v`):
- `TestTwoNodeDirect` — no NAT → direct path (`curAddr` populated).
- `TestSymmetricNATFallback` — symmetric/symmetric NAT → DERP-only
  (`curAddr=""`, `relay` set), ping still transits. **The GATE.**
- `TestEasyNATDirect` — endpoint-independent NAT → direct hole-punch
  (`curAddr` populated). Proves the engine upgrades to direct when possible.

Conclusion: Strategy A is viable. The engine + magicsock + DERP work behind an
external package using only exported API + a hand-built netmap. mesh-signal only
has to be "an extremely stripped-down control plane" (tailscale's own words for
the `meshStacks` test helper we ported).

## The netmap shape (what magicsock actually reads)
Per node, mesh-signal builds one `*netmap.NetworkMap`:

```
NetworkMap{
  NodeKey:  <self node public key>          // key.NodePublic
  SelfNode: tailcfg.Node{                    // .View()
    Addresses: [ <self mesh IP>/32 ]         // []netip.Prefix
  }.View()
  Peers: [ for each OTHER peer:              // []tailcfg.NodeView
    tailcfg.Node{
      ID:         <stable peer id>           // tailcfg.NodeID
      Name:       <peer name>                // for logs/MagicDNS
      Key:        <peer node public key>     // key.NodePublic
      DiscoKey:   <peer disco public key>    // key.DiscoPublic (from conn.DiscoPublicKey())
      Addresses:  [ <peer mesh IP>/32 ]
      AllowedIPs: [ <peer mesh IP>/32, ...subnets ]  // Phase 5 adds advertised CIDRs here
      Endpoints:  [ <peer ip:port>, ... ]    // reflexive/STUN candidates, []netip.AddrPort
      HomeDERP:   <peer DERP region id>      // int; the relay fallback anchor
    }.View()
  ]
}
```

Mapping from our registry (`MeshPeer`):
| netmap field | MeshPeer source |
|--------------|-----------------|
| Key | `public_key` |
| DiscoKey | reported by node at enroll (engine `DiscoPublicKey()`) |
| Addresses / AllowedIPs[0] | `mesh_ip` |
| AllowedIPs[1..] | `advertised_subnets` (Phase 5) |
| Endpoints | `endpoints[]` (from Hello/heartbeat) |
| HomeDERP | DERP region assigned by mesh-signal |
| ID / Name | MeshPeer id / node label |

## DERP map (what mesh-signal pushes)
A `*tailcfg.DERPMap` of our `apps/derp` regions:
```
DERPMap{ Regions: { <id>: DERPRegion{
  RegionID, RegionCode,
  Nodes: [ DERPNode{ Name, RegionID, HostName, IPv4, STUNPort, DERPPort,
                     /* prod: real TLS, not InsecureForTests */ } ]
}}}
```
Each peer's `HomeDERP` references a region id here.

## Engine API surface (the vendoring blast radius)
Exported tailscale symbols the agent engine depends on — pin these:

**magicsock** (`tailscale.com/wgengine/magicsock`)
- `NewConn(Options{NetMon, EventBus, Metrics, Logf, HealthTracker,
  DisablePortMapper, EndpointsFunc, [TestOnlyPacketListener]}) (*Conn, error)`
- `conn.SetDERPMap(*tailcfg.DERPMap)`
- `conn.SetPrivateKey(key.NodePrivate) error`
- `conn.Bind() conn.Bind`
- `conn.DiscoPublicKey() key.DiscoPublic`
- `conn.SetNetworkMap(self tailcfg.NodeView, peers []tailcfg.NodeView)`
- `conn.UpdatePeers(set.Set[key.NodePublic])`
- `conn.UpdateStatus(*ipnstate.StatusBuilder)` → path observability
  (`PeerStatus.CurAddr` = direct addr, `.Relay` = DERP region)

**wgcfg** (`tailscale.com/wgengine/wgcfg`)
- `NewDevice(tun.Device, conn.Bind, *device.Logger) *device.Device`
- `ReconfigDevice(*device.Device, *Config, logger.Logf) error`

**nmcfg** (`tailscale.com/wgengine/wgcfg/nmcfg`)
- `WGCfg(key.NodePrivate, *netmap.NetworkMap, logger.Logf,
  netmap.WGConfigFlags, tailcfg.StableNodeID) (*wgcfg.Config, error)`

**device** (tailscale fork — see pins) — `SetPeerByIPPacketFunc`, `SetPrivateKey`
**keys** — `key.NewNode()`, `.Public()`, `key.NodePrivateAs[...]`
**DERP server** — `tailscale.com/derp/derpserver`: `New`, `Handler`, `ProbeHandler`

## Critical vendoring pins (API drift found in spike)
- Go **≥ 1.26.4** (tailscale v1.100.0 requires it).
- `tailscale.com v1.100.0`.
- WireGuard is tailscale's **fork**: `github.com/tailscale/wireguard-go`
  (NOT `golang.zx2c4.com/wireguard` — wrong one fails with a `tun.Device`
  interface mismatch).
- Package moves vs older docs: DERP server `tailscale.com/derp` →
  `tailscale.com/derp/derpserver`; `net/nettype` → `types/nettype`.
- Isolate ALL of the above behind `apps/agent/internal/mesh` so a tailscale
  bump is a contained edit.

## What the spike did NOT prove (carry into prod Phase 4)
- Runtime path migration on the SAME tunnel (direct → drop → DERP → re-upgrade).
  Spike proved the two endpoints (direct works; DERP works when direct is
  impossible); live flap-and-recover is a magicsock internal we rely on but did
  not exercise. Validate in a prod integration test.
- Real step-ca TLS on DERP (spike used self-signed / InsecureForTests).
- Multi-region DERP selection + HA.
- Performance (userspace-WG throughput) — benchmark before DB-replication paths.
- MTU under the engine (and double-encap for Phase 6 k8s).
