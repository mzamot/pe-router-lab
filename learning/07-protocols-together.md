# Part 7 — How All Protocols Work Together

Now that you understand each protocol individually, let's see how they form a
cohesive system. Each protocol has a specific role, and they integrate in a
specific order.

## 7.1 The Protocol Stack (Bottom to Top)

```
┌─────────────────────────────────────────────────────────────┐
│  Application Layer                                          │
│  Compute workloads, machine network, node communication     │
│  "I need to send a packet to 10.55.99.8"                    │
├─────────────────────────────────────────────────────────────┤
│  L2 Data Plane: VXLAN                                       │
│  Encapsulates Ethernet frames in UDP/IP tunnels             │
│  "I'll wrap this frame and send it to VTEP 10.55.96.154"    │
├─────────────────────────────────────────────────────────────┤
│  L2 Control Plane: BGP EVPN (AS 65501)                      │
│  Distributes MAC/IP info between VTEPs                      │
│  "MAC aa:bb:cc is behind VTEP 10.55.96.154 on VNI 330"     │
├─────────────────────────────────────────────────────────────┤
│  L3 VPN Control Plane: BGP L3VPN (AS 65500)                 │
│  Distributes VRF routes (including VTEP subnets)            │
│  "Subnet 10.55.96.152/30 is on PE2, SRv6 SID fd00:30:17:3" │
├─────────────────────────────────────────────────────────────┤
│  Transport: SRv6                                            │
│  Encapsulates VRF traffic in IPv6 for fabric transit        │
│  "Wrap this packet with dst=fd00:30:17:3:: for delivery"    │
├─────────────────────────────────────────────────────────────┤
│  Underlay Routing: IS-IS                                    │
│  Provides IPv6 reachability between all nodes               │
│  "fd00:30:17::/48 is reachable via eth1 toward spine"       │
├─────────────────────────────────────────────────────────────┤
│  Physical: Ethernet + fiber                                 │
│  Moves bits between PE and spine                            │
│  "Here are the electrical signals for this frame"           │
└─────────────────────────────────────────────────────────────┘
```

## 7.2 Startup Order (Why Order Matters)

When the system boots, protocols must start in order because each depends on
the layer below:

1. **Physical links come up** — eth1 interfaces are active
2. **IS-IS forms adjacencies** (~10 seconds) — loopbacks and SRv6 locators
  become reachable
3. **BGP L3VPN sessions establish** (~15 seconds after IS-IS) — PEs exchange
  VRF routes, including VTEP subnets
4. **VTEP reachability is established** — l2-vxlan namespaces can now reach
  each other through CMN VRF → SRv6 → fabric
5. **BGP EVPN sessions establish** (~10 seconds after VTEP reachability) — PEs
  exchange MAC/IP information
6. **VXLAN tunnels are operational** — FDB entries installed, BUM flooding
  configured
7. **Machine network is connected** — hosts can ARP and communicate at L2

If step N fails, steps N+1 through 7 all fail. This is why debugging starts
from the bottom up.

## 7.3 How Protocols Interact

**IS-IS ↔ SRv6:** IS-IS carries SRv6 locator prefixes in its TLV extensions.
When IS-IS advertises `fd00:30:16::/48`, it's simultaneously providing
underlay reachability AND SRv6 forwarding information. Transit routers don't
need to know these are SRv6 locators — they just route IPv6 normally.

**IS-IS ↔ BGP:** IS-IS provides the transport for BGP sessions. BGP peers
using loopback addresses (e.g., `fc00:0:0:1::1` ↔ `fc00:0:0:1::16`). IS-IS
ensures these addresses are reachable. Without IS-IS routes, BGP TCP sessions
can't form.

**BGP L3VPN ↔ SRv6:** BGP VPN routes carry SRv6 SIDs as attributes. When PE1
advertises a CMN route, the BGP update includes the SRv6 SID
(`fd00:30:16:3::`) that tells remote PEs how to encapsulate traffic destined
for that VRF. The receiving PE installs a route with `seg6` encapsulation,
linking the VPN control plane to the SRv6 data plane.

**BGP L3VPN ↔ BGP EVPN:** L3VPN distributes the CMN VRF routes that include
VTEP subnets. EVPN sessions run over VTEP IPs. So L3VPN provides the
transport for EVPN — a meta-dependency where one BGP instance enables another.

**EVPN ↔ VXLAN:** EVPN is the control plane; VXLAN is the data plane. EVPN
tells the bridge's FDB where each MAC is (which VTEP). VXLAN handles the
actual encapsulation/decapsulation of frames. Without EVPN, VXLAN would fall
back to flood-and-learn. Without VXLAN, EVPN's MAC information would have no
data plane to drive.

## 7.4 Summary: Each Protocol's Essential Contribution


| Protocol  | What It Provides                                  | What Breaks Without It                                                                    |
| --------- | ------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| IS-IS     | Underlay reachability (loopbacks + SRv6 locators) | Everything — no routing = no communication                                                |
| SRv6      | VRF-aware transport encapsulation                 | VRF traffic can't cross the fabric                                                        |
| BGP L3VPN | VRF route distribution + SRv6 SID mapping         | PEs don't know about each other's VRF subnets; VTEP reachability fails                    |
| BGP EVPN  | MAC/IP learning without flooding                  | VXLAN falls back to flood-and-learn (doesn't scale, may not work at all in this topology) |
| VXLAN     | L2 tunneling over L3                              | No Layer 2 connectivity between nodes on different PEs                                    |


---

# Quick Reference: All Verification Commands

## From the containerlab host

```bash
# Spine FRR shell:
sudo podman exec clab-srv6-lab-spine vtysh

# Inside vtysh:
  show isis neighbor
  show isis route level-1
  show ipv6 route isis
  show bgp ipv4 vpn summary
  show bgp ipv4 vpn
```

## From inside a PE VM

```bash
# SSH in:
ssh clab@clab-srv6-lab-pe1   # password: clab@123

# pe-router FRR shell:
sudo podman exec frr-pe-router vtysh

# Inside vtysh:
  show isis neighbor
  show bgp ipv4 vpn summary
  show bgp ipv4 vpn
  show ip route vrf CMN
  show segment-routing srv6 locator
  show segment-routing srv6 sid

# l2-vxlan FRR shell:
sudo podman exec frr-l2-vxlan vtysh

# Inside vtysh:
  show bgp l2vpn evpn summary
  show bgp l2vpn evpn
  show evpn vni
  show evpn mac vni 330

# Linux kernel inspection:
sudo ip netns exec pe-router ip route show vrf CMN
sudo ip netns exec pe-router ip -6 route show | grep seg6
sudo ip netns exec l2-vxlan bridge fdb show dev vxlan0
sudo ip netns exec l2-vxlan bridge vlan show
sudo ip netns exec l2-vxlan ip -d link show vxlan0

# Connectivity tests:
ping 10.55.99.7    # PE1 machine network
ping 10.55.99.8    # PE2 machine network
ping 10.55.99.9    # PE3 machine network
```


---

[← Part 6: Packet Walk](06-packet-walk.md) | [Back to Index](../LEARNING.md)
