# Part 6 — Full Packet Walk

Let's trace a ping from PE1 (`10.55.99.7`) to PE2 (`10.55.99.8`). This
exercises every layer of the stack. Follow along with the architecture diagram
from Section 1.5.

## 6.1 Step 1: Default Namespace → L2 VXLAN Bridge

```
PE1 default namespace:
  Source: 10.55.99.7 (on ve-home-l2-br)
  Destination: 10.55.99.8

  Routing table lookup:
    10.55.99.0/26 is connected on ve-home-l2-br
    → Same subnet! Use L2 (ARP) instead of routing.
    → ARP for 10.55.99.8 on ve-home-l2-br
```

Because 10.55.99.8 is in the same `/26` subnet as 10.55.99.7, the kernel knows
it's directly reachable at Layer 2. It doesn't need a router — it just needs
the destination's MAC address. So it sends an ARP request: "who has 10.55.99.8?
Tell 10.55.99.7."

The ARP request (a broadcast frame) enters the l2-vxlan namespace via the
`ve-home-l2-br` ↔ `ve-l2-br-home` veth pair. `ve-l2-br-home` is a bridge
port on `br0`, so the frame enters the bridge.

## 6.2 Step 2: Bridge Lookup + VXLAN Encapsulation

```
l2-vxlan namespace (br0):
  The bridge has an FDB entry (from EVPN Type-2):
    <PE2's ve-home-l2-br MAC> → VTEP 10.55.96.154

  VXLAN encapsulation:
    Inner: original Ethernet frame (ICMP echo request)
    Outer IP: src=10.55.96.150  dst=10.55.96.154
    Outer UDP: dst port 4789
    VNI: 330
```

If the bridge already has the destination MAC in its FDB (learned from EVPN),
it sends the frame directly to the correct VTEP. If not, it floods to all
VTEPs (using the Type-3 BUM entries). In either case, the frame exits via
`vxlan0`, which adds the VXLAN headers.

## 6.3 Step 3: VTEP → PE-Router CMN VRF

The VXLAN-encapped packet exits via the default route in l2-vxlan:

```
l2-vxlan routing table:
  default via 10.55.96.149 dev ve-l2-pe-cmn

  Packet goes through veth:
    ve-l2-pe-cmn (l2-vxlan) → ve-pe-cmn-l2 (pe-router, CMN VRF)
```

The l2-vxlan namespace has only one route: `default via 10.55.96.149`. All
traffic leaves via this gateway, which is the pe-router's CMN VRF interface
on the other end of the veth pair.

## 6.4 Step 4: CMN VRF → SRv6 Encapsulation

```
pe-router CMN VRF (table 4):
  Destination: 10.55.96.154
  Route: 10.55.96.152/30 via fc00:0:0:1::17, seg6 fd00:30:17:3::

  SRv6 encapsulation:
    Outer IPv6: src=fd00:30:16::  dst=fd00:30:17:3::
    Inner: the entire VXLAN packet from step 2
```

The packet is now **double-encapsulated**: SRv6 wrapping VXLAN wrapping the
original ICMP. The SRv6 destination address (`fd00:30:17:3::`) tells every
transit router where to forward (PE2's locator) AND tells PE2 what to do when
it arrives (decapsulate into VRF CMN).

## 6.5 Step 5: Fabric Transit (Spine)

```
pe-router default table (underlay):
  Destination: fd00:30:17:3:: matches fd00:30:17::/48
  Route: via fe80::... (spine's link-local), dev eth1

  → Packet sent to the spine over the IS-IS fabric link

spine:
  Destination: fd00:30:17::/48 → via eth2 (toward PE2)
  The spine is a pure transit router — no decapsulation, no SRv6 processing
  It just sees a normal IPv6 packet and forwards it

  → Packet forwarded to PE2
```

## 6.6 Step 6: SRv6 Decapsulation at PE2

```
PE2 pe-router:
  Destination: fd00:30:17:3:: is a local SRv6 SID
  Action: End.DT4 → decapsulate, deliver to VRF table 4 (CMN)

  After decap: the original VXLAN packet
    Destination: 10.55.96.154 (local on ve-pe-cmn-l2)
```

## 6.7 Step 7: CMN VRF → L2 VXLAN Namespace

```
PE2 CMN VRF:
  10.55.96.154 is connected on ve-pe-cmn-l2
  → Deliver to ve-pe-cmn-l2

  Packet crosses veth:
    ve-pe-cmn-l2 (pe-router) → ve-l2-pe-cmn (l2-vxlan)
```

## 6.8 Step 8: VXLAN Decapsulation + Bridge Delivery

```
PE2 l2-vxlan:
  vxlan0 receives the packet (dst port 4789, VNI 330)
  Strips VXLAN header → inner Ethernet frame
  Bridge lookup → forward to ve-l2-br-home (VLAN 30)

  Packet crosses veth:
    ve-l2-br-home (l2-vxlan) → ve-home-l2-br (default namespace)
```

## 6.9 Step 9: Arrival

```
PE2 default namespace:
  ICMP echo request arrives on ve-home-l2-br
  Destination: 10.55.99.8 (local address)
  → Generate ICMP echo reply
  → The reply takes the reverse path (9 steps back to PE1)
```

## 6.10 Watching It Live

```bash
# On PE1 — capture the SRv6-encapped packets leaving toward the spine:
sudo ip netns exec pe-router tcpdump -i eth1 -nn ip6 -c 5

# On the spine — watch transit traffic:
sudo podman exec clab-srv6-lab-spine tcpdump -i eth1 -nn ip6 -c 5

# On PE1 — capture VXLAN packets in the l2-vxlan namespace:
sudo ip netns exec l2-vxlan tcpdump -i ve-l2-pe-cmn -nn udp port 4789 -c 5

# On PE1 — see the original ICMP inside the bridge:
sudo ip netns exec l2-vxlan tcpdump -i ve-l2-br-home -nn icmp -c 5

# While those tcpdumps are running, from another terminal:
ping -c 3 10.55.99.8
```

## 6.11 The Encapsulation Stack

For a single ICMP ping traversing the fabric:

```
┌─────────────────────────────────────────────────────────────┐
│ Outer Ethernet (L2 between PE1 and spine)         14 bytes  │
├─────────────────────────────────────────────────────────────┤
│ SRv6 IPv6 Header                                 40 bytes  │
│   src: fd00:30:16::                                         │
│   dst: fd00:30:17:3::                                       │
├─────────────────────────────────────────────────────────────┤
│ VXLAN Outer IPv4 Header                          20 bytes  │
│   src: 10.55.96.150 (PE1 VTEP)                              │
│   dst: 10.55.96.154 (PE2 VTEP)                              │
├─────────────────────────────────────────────────────────────┤
│ VXLAN Outer UDP Header                            8 bytes  │
│   dst port: 4789                                            │
├─────────────────────────────────────────────────────────────┤
│ VXLAN Header                                      8 bytes  │
│   VNI: 330                                                  │
├─────────────────────────────────────────────────────────────┤
│ Inner Ethernet                                   14 bytes  │
├─────────────────────────────────────────────────────────────┤
│ Inner IPv4                                       20 bytes  │
│   src: 10.55.99.7  dst: 10.55.99.8                         │
├─────────────────────────────────────────────────────────────┤
│ ICMP Echo Request                                 8 bytes  │
├─────────────────────────────────────────────────────────────┤
│ Payload                                          56 bytes  │
└─────────────────────────────────────────────────────────────┘

Total overhead: 40 (SRv6) + 50 (VXLAN) = 90 bytes
Total on wire: 56 + 8 + 20 + 14 + 8 + 8 + 20 + 40 + 14 = 188 bytes
```

This is why the production system uses MTU 9000 — with 90 bytes of
encapsulation overhead, a 1500-byte inner frame becomes 1590 bytes on the
fabric link.


---

[← Part 5: VXLAN & EVPN](05-vxlan-evpn.md) | [Back to Index](../LEARNING.md) | [Part 7: Protocols Together →](07-protocols-together.md)
