# Part 5 — VXLAN and BGP EVPN

## 5.0 What is VXLAN? What is EVPN? (The Big Picture)

Just as we explained L3VPN before diving into its mechanics, let's understand
what VXLAN and EVPN are, where they come from, and why this architecture needs
them.

### What is VXLAN?

**VXLAN (Virtual eXtensible LAN)** is a network tunneling protocol that wraps
entire Ethernet frames inside UDP/IP packets. It was developed around 2011-2012
by VMware, Cisco, and Arista (published as RFC 7348 in 2014) to solve
fundamental problems with traditional Layer 2 networking in large-scale data
centers.

To understand why VXLAN exists, you first need to understand the problem it
solves. Think about how a traditional Ethernet network works. You have physical
switches connected by cables. Devices plugged into the same switch (or
switches connected together) are in the same **broadcast domain** — they can
send Ethernet frames directly to each other using MAC addresses. If you want
separate broadcast domains, you use **VLANs** (802.1Q), which partition a
switch into multiple virtual switches, each with its own broadcast domain.

This works fine for small networks. But it has three critical limitations at
data center scale:

1. **VLAN scalability:** The 802.1Q VLAN ID is only 12 bits, giving you a
  maximum of 4,094 VLANs. In a cloud environment with thousands of tenants
   each needing multiple isolated networks, 4,094 is nowhere near enough.
2. **Physical topology constraints:** Traditional L2 networks require all
  switches in a broadcast domain to be connected in a spanning-tree topology
   (to prevent loops). Spanning tree blocks redundant links, wasting bandwidth,
   and limits the network to a tree shape. Modern data centers want a highly
   redundant **leaf-spine** topology with multiple active paths — but that
   requires L3 routing between switches, which breaks L2 connectivity.
3. **Geographic limitations:** A VLAN can only span switches that are directly
  connected (or connected through other L2 switches). You can't stretch a VLAN
   across an IP-routed network. If your servers are in different racks connected
   by L3 routers, they can't be in the same VLAN.

VXLAN solves all three problems with a single elegant idea: **wrap the entire
Ethernet frame in a UDP/IP packet and send it across the IP network**.

1. **24-bit VNI:** Instead of 4,094 VLANs, VXLAN provides over 16 million
  virtual network identifiers (VNIs). Each VNI is a separate broadcast domain.
2. **L2 over L3:** By encapsulating Ethernet frames in UDP/IP packets, VXLAN
  allows L2 traffic to traverse any IP network. The physical network can be a
   pure L3 leaf-spine fabric with ECMP routing, while the logical network
   provides any L2 topology the applications need.
3. **Location independence:** Since VXLAN rides on IP, it works across any IP
  network — even across cities or continents. Servers in different racks,
   different data centers, or even different countries can be in the same VXLAN
   segment.

### Key VXLAN Concepts

Before we walk through how VXLAN works, let's define the key terms:

- **VNI (VXLAN Network Identifier):** A 24-bit number (0 to ~16 million) that
identifies a virtual network. All devices in the same VNI are in the same
broadcast domain, like a VLAN ID but with far more capacity. In this lab,
we use VNI 330 for the machine network.
- **VTEP (VXLAN Tunnel Endpoint):** The device that performs VXLAN
encapsulation and decapsulation. VTEPs sit at the edge of the VXLAN network.
In hardware environments, a VTEP is usually a top-of-rack (leaf) switch. In
software environments (like ours), a VTEP is a Linux `vxlan` device. Each
VTEP has an IP address that other VTEPs use to send encapsulated traffic to
it. In this lab, the VTEP is the `vxlan0` device in each PE's l2-vxlan
namespace.
- **FDB (Forwarding Database):** A table that maps MAC addresses to VTEPs. When
a VTEP receives a frame destined for a MAC address, it looks up the MAC in
its FDB to find which remote VTEP hosts that MAC. The FDB is populated either
by flood-and-learn (the old way) or by EVPN (the modern, scalable way).
- **UDP port 4789:** The standard VXLAN destination port. When a VTEP
encapsulates a frame, the outer UDP destination port is always 4789. The
source port is typically a hash of the inner frame's header fields — this
enables ECMP load balancing across multiple paths in the IP underlay, because
routers can hash on the outer 5-tuple.
- **Inner frame vs. outer packet:** The "inner frame" is the original Ethernet
frame from the host — it travels unchanged through the VXLAN tunnel. The
"outer packet" is the UDP/IP wrapper added by the VTEP. The outer IP
addresses are the VTEP IPs (not the host IPs), and the outer UDP port is

### How VXLAN Works (Step by Step)

Here's what happens when Host A (on PE1) sends a frame to Host B (on PE2):

```
Step 1: Host A sends a normal Ethernet frame
   ┌──────────────────────────────────────┐
   │ dst MAC: BB:BB:BB:BB:BB:BB           │
   │ src MAC: AA:AA:AA:AA:AA:AA           │
   │ Payload: "Hello, Host B"             │
   └──────────────────────────────────────┘
   This frame arrives at PE1's bridge (br0) via the host's veth pair.

Step 2: The bridge looks up the destination MAC in its FDB
   The FDB (Forwarding Database) says:
     BB:BB:BB:BB:BB:BB → VTEP 10.55.96.154 (PE2)
   (This entry was learned from EVPN, or if unknown, the bridge floods
    to all VTEPs.)

Step 3: The bridge sends the frame to vxlan0 (the VTEP device)
   vxlan0 wraps the frame:
   ┌──────────────────────────────────────┐
   │ Outer IP header:                     │
   │   src = 10.55.96.150 (local VTEP)    │
   │   dst = 10.55.96.154 (remote VTEP)   │
   │ Outer UDP header:                    │
   │   src port = (hash of inner frame)   │
   │   dst port = 4789                    │
   │ VXLAN header:                        │
   │   VNI = 330                          │
   │ ────────────────────────────────     │
   │ Original Ethernet frame (unchanged)  │
   └──────────────────────────────────────┘
   The outer IP addresses are the VTEP IPs, NOT the host IPs.
   The VNI identifies which virtual network this frame belongs to.

Step 4: The encapsulated packet is routed to the remote VTEP
   In this architecture, it goes:
     l2-vxlan → CMN VRF → SRv6 encap → IS-IS underlay → PE2
   But from VXLAN's perspective, it's just an IP packet being routed.

Step 5: PE2's vxlan0 receives and decapsulates the packet
   PE2's VTEP sees dst IP = its own VTEP address, dst port = 4789.
   It strips the outer headers and recovers the original Ethernet frame.
   The frame is delivered to the bridge, which forwards it to Host B
   via the appropriate bridge port.
```

From Host A and Host B's perspective, this is completely transparent. They
think they're on the same Ethernet segment. They don't know about VXLAN, VTEPs,
VNIs, or the IP fabric between them. This is the entire point — VXLAN creates
the **illusion** of direct L2 connectivity over an L3 routed network.

### VXLAN Without a Control Plane (Flood-and-Learn)

Before EVPN existed, VXLAN had a significant limitation: VTEPs had no way to
know which remote VTEP hosted a given MAC address. The solution was
**flood-and-learn**, which works exactly like a traditional Ethernet switch:

1. When a VTEP receives a frame for an unknown MAC, it encapsulates and sends
  a copy to **every** remote VTEP (flooding).
2. When the destination host responds, the reply comes back through a specific
  VTEP, and the originating VTEP learns the MAC → VTEP mapping.
3. Future frames for that MAC go directly to the correct VTEP (until the entry
  ages out and flooding happens again).

This works, but it has serious problems at scale:

- **Bandwidth waste:** With 100 VTEPs, every unknown MAC lookup generates 99
copies of the frame.
- **ARP storms:** ARP is broadcast by nature, so every ARP request is flooded
to all VTEPs. In a busy network, this is a constant stream of flooded
traffic.
- **No central visibility:** No way to query "where is MAC X?" — you just have
to wait for floods and learn.

This is where EVPN comes in — it replaces flood-and-learn with a BGP-based
control plane, which we'll cover next.

### Where VXLAN Is Normally Used

VXLAN is deployed in virtually every modern data center:

- **Cloud providers (AWS, Azure, GCP):** Each customer's VPC (Virtual Private
Cloud) is implemented as one or more VXLAN segments. When you create a VPC
subnet in AWS, you're creating a VXLAN VNI behind the scenes. Your EC2
instances communicate at L2 within the VPC, even though they may be on
different physical hosts in different racks.
- **Enterprise data centers:** Companies running VMware vSphere, Nutanix, or
other hyperconverged infrastructure use VXLAN to provide network
virtualization. VMs can migrate between physical hosts (vMotion) without
changing IP or MAC addresses, because VXLAN extends the L2 segment to
wherever the VM moves.
- **Leaf-spine fabrics:** Modern data center networks use a leaf-spine
architecture where every leaf switch connects to every spine switch via L3
routing (BGP or OSPF). VXLAN overlays on top of this L3 fabric provide the
L2 connectivity that applications need, while the L3 underlay provides fast,
redundant, ECMP-capable routing.
- **Multi-site deployments:** Stretching L2 networks between data centers in
different locations. A database cluster that requires L2 adjacency between
its nodes can span two data centers connected by a WAN.
- **Container platforms:** Some CNI plugins (like Flannel in VXLAN mode) use
VXLAN to create a pod network overlay. In this architecture, VXLAN provides
the machine network (node-to-node) connectivity rather than the pod network.

### Why THIS Architecture Uses VXLAN

In this architecture, the three PEs are connected through an SRv6 IP fabric.
They can exchange IP traffic via L3VPN. But the machine network requires all
nodes to be in the same Layer 2 broadcast domain — they must be
able to ARP for each other, which is a Layer 2 operation.

VXLAN bridges this gap. It takes the L2 traffic from the machine network
(`10.55.99.0/26`) and tunnels it through the L3 fabric. Each PE has a VTEP in
its l2-vxlan namespace. When a node sends an ARP request or a data frame to
another node, VXLAN encapsulates it and delivers it to the correct remote VTEP
via the CMN VRF → SRv6 path.

Without VXLAN, we'd need either:

- **Direct L2 connectivity** between all PEs (physical cables carrying VLANs) —
defeats the purpose of the SRv6 fabric
- **Proxy ARP + L3 routing for everything** — complex, fragile, and breaks
applications that assume L2 adjacency
- **A different overlay like GRE or GENEVE** — possible, but VXLAN + EVPN is
the industry standard with the best tooling and support

### What is EVPN? (The Control Plane for VXLAN)

Now that you understand VXLAN as the data plane tunnel, the natural question is:
**how does a VTEP know which remote VTEP to send frames to?** That's what EVPN
solves.

### The Difference Between L3VPN and EVPN

You just learned about **L3VPN**, which provides Layer 3 (IP routing) isolation
between VRFs. L3VPN is powerful, but it only carries **IP prefixes** — it tells
routers "subnet X is reachable via PE Y." It does NOT carry Layer 2 information
like MAC addresses or preserve Ethernet broadcast domains.

**EVPN (Ethernet Virtual Private Network)** is the Layer 2 counterpart. While
L3VPN distributes IP routes between VRFs, EVPN distributes **MAC addresses and
ARP/IP bindings** between virtual switches (VTEPs). EVPN tells VTEPs "MAC
address `aa:bb:cc:11:22:33` is behind VTEP `10.55.96.154` on VNI 330."

Both L3VPN and EVPN use BGP as their distribution mechanism — but they use
different BGP address families:

- L3VPN uses **IPv4/IPv6 VPN** address families (carries IP prefixes + VRF
metadata)
- EVPN uses the **L2VPN EVPN** address family (carries MAC/IP bindings + VTEP
metadata)

**VXLAN** is NOT a control plane protocol — it's a **data plane tunneling
protocol**. VXLAN wraps Ethernet frames in UDP/IP packets so they can traverse
an IP network. Think of VXLAN as the "truck" that carries the L2 frames, and
EVPN as the "logistics system" that tells each truck where to go.

You can run VXLAN without EVPN (using flood-and-learn, as described above), but
it doesn't scale. You can conceptually run EVPN without VXLAN (using other
tunneling methods like MPLS or GRE), but VXLAN is the standard pairing. In this
architecture, EVPN + VXLAN work together: EVPN provides the intelligence, VXLAN
provides the transport.

### The Problem EVPN Was Invented to Solve

Traditional Layer 2 networking has a fundamental scaling problem. In a physical
Ethernet network, when a switch doesn't know where a MAC address is, it
**floods** the frame to every port. Every switch in the broadcast domain
receives it. This worked fine for small networks, but in large data centers
with thousands of hosts and hundreds of switches, flooding consumes enormous
bandwidth and CPU.

Early VXLAN deployments inherited this problem. When a VTEP didn't know which
remote VTEP a MAC was behind, it flooded the encapsulated frame to ALL remote
VTEPs. With 100 VTEPs, that's 99 copies of every unknown frame. Add ARP
(which is broadcast by nature), and you have a constant background storm of
flooded traffic.

**EVPN was invented to replace flood-and-learn with a BGP-based control
plane.** Instead of flooding to discover MACs, VTEPs use BGP to advertise their
learned MACs to each other proactively:

1. A host sends a frame to its local VTEP
2. The VTEP learns the host's MAC and tells BGP about it
3. BGP advertises the MAC (as an EVPN Type-2 route) to all other VTEPs
4. Remote VTEPs install the MAC → VTEP mapping in their forwarding database
5. When traffic needs to reach that MAC, the remote VTEP sends it directly to
  the correct VTEP — no flooding needed

This transforms L2 networking from a "shout and listen" model (flooding) to a
"look up and send" model (control plane), making it scale to data center size.

### Where EVPN Is Normally Used

EVPN has become the **standard** for data center overlay networking:

- **Data center fabrics:** Every modern data center fabric (Cisco ACI, Arista
CloudVision, Cumulus/NVIDIA, SONiC) uses EVPN + VXLAN to provide L2
connectivity across a leaf-spine IP fabric. Servers on different racks
(different leaf switches) can be in the same L2 segment.
- **Data center interconnect (DCI):** Stretching L2 networks between data
centers in different cities. A VM on Rack A in Data Center 1 can
communicate at L2 with a VM on Rack B in Data Center 2, hundreds of miles
away.
- **Multi-tenant environments:** Cloud providers use EVPN to isolate tenant
L2 networks. Each tenant gets their own VNI(s), and EVPN ensures MAC
addresses are only advertised to VTEPs that serve that tenant.
- **Campus networks:** Extending VLANs across campus buildings connected by an
IP backbone.
- **Service provider L2VPN services:** SPs offer "Virtual Private LAN Service"
to enterprises that need L2 connectivity between sites.

EVPN is defined in RFC 7432 and has been widely deployed since ~2015. Like
L3VPN, it's a mature, proven technology.

### What EVPN + VXLAN Do in THIS Architecture

In this architecture, EVPN + VXLAN provide the **Layer 2 machine network** that
the compute nodes require:

- **VXLAN** tunnels Ethernet frames between the l2-vxlan namespaces on each PE.
When PE1's host (10.55.99.7) sends a frame to PE2's host (10.55.99.8), VXLAN
encapsulates it and sends it from VTEP 10.55.96.150 to VTEP 10.55.96.154.
- **EVPN** tells each VTEP where every MAC address is, so VXLAN can send frames
directly to the right VTEP without flooding. EVPN also advertises VTEP
membership for BUM (broadcast/unknown/multicast) traffic, so broadcast frames
(like ARP requests) reach all VTEPs that participate in VNI 330.
- **Together**, they make the machine network (`10.55.99.0/26`) appear as a
single Ethernet segment spanning all three PEs. Any host can ARP for any
other host, exactly as if they were plugged into the same physical switch.

**Relationship to L3VPN:** EVPN sessions (BGP in the l2-vxlan namespace) run
over VTEP IPs, which are in the CMN VRF. The CMN VRF routes are distributed by
L3VPN. So L3VPN provides the transport for EVPN — they're not competing
technologies, they're complementary layers:

```
L3VPN says: "PE2's VTEP subnet 10.55.96.152/30 is reachable via SRv6 SID
            fd00:30:17:3::"
            → This makes PE2's VTEP IP reachable from PE1

EVPN says:  "MAC aa:bb:cc:11:22:33 is behind VTEP 10.55.96.154 on VNI 330"
            → This tells PE1 where to send frames for that MAC

VXLAN does: Encapsulates the frame and sends it to VTEP 10.55.96.154
            → The encapsulated packet traverses CMN VRF → SRv6 → underlay
```

## 5.0a Why L2 Connectivity Matters (In This Architecture)

Up to this point, we have a working L3 VPN: every PE's CMN VRF can reach every
other PE's CMN subnets via SRv6. But the machine network requires something
more — **Layer 2 connectivity** between nodes. Specifically, these scenarios
require L2:

- **ARP resolution between nodes** — When a node wants to send a packet to
another node on the same subnet, it first ARPs for the destination's MAC
address. ARP is a Layer 2 broadcast protocol. Without L2 connectivity, ARP
requests can't reach remote nodes.
- **Cluster-internal services that assume L2 adjacency** — Some cluster
networking features (like certain CNI plugins and load balancers) assume all
nodes in a subnet can communicate at L2.
- **The machine network (`10.55.99.0/26`) where nodes communicate** — This is
the primary network for inter-node communication. All nodes must be in the
same `/26` subnet and able to ARP for each other.

Without VXLAN, nodes on different physical servers couldn't be in the same
L2 domain — they're separated by an L3 routed network (the SRv6 fabric). Each
PE's `10.55.99.X` address would be unreachable from other PEs at Layer 2.
VXLAN stretches a single broadcast domain across the entire fabric, making all
three PEs appear to be on the same Ethernet segment.

## 5.1 VXLAN Encapsulation Details

Now that you understand what VXLAN is and why it exists, let's look at the
exact encapsulation format. Understanding the wire format helps when reading
tcpdump output or debugging connectivity issues.

**How VXLAN encapsulation works:**

```
Original Ethernet frame (inside the bridge):
┌──────────────────────────────────────────┐
│ dst MAC: aa:bb:cc:11:22:33               │
│ src MAC: aa:bb:cc:44:55:66               │
│ Ethertype: 0x0800 (IPv4)                 │
│ IP: src=10.55.99.7  dst=10.55.99.8       │
│ ICMP Echo Request                        │
└──────────────────────────────────────────┘

After VXLAN encapsulation:
┌───────────────────────────────────────────┐
│ Outer Ethernet (14 bytes)                 │  ← L2 header for the next hop
├───────────────────────────────────────────┤
│ Outer IP (20 bytes)                       │  ← IP header for VTEP-to-VTEP
│   src = 10.55.96.150 (local VTEP)         │     routing
│   dst = 10.55.96.154 (remote VTEP)        │
├───────────────────────────────────────────┤
│ Outer UDP (8 bytes)                       │  ← UDP wrapper (VXLAN uses UDP
│   src port = hash-based    dst = 4789     │     for NAT traversal and ECMP)
├───────────────────────────────────────────┤
│ VXLAN Header (8 bytes)                    │  ← identifies the virtual network
│   VNI = 330                               │
├───────────────────────────────────────────┤
│ Inner Ethernet frame (original L2 frame)  │  ← the actual traffic being
│   The complete original frame above       │     tunneled, unchanged
└───────────────────────────────────────────┘
```

Total overhead: ~50 bytes (14 outer Ethernet + 20 outer IP + 8 UDP + 8 VXLAN).
With the SRv6 encapsulation on top of that, total overhead is ~90 bytes. This
is why jumbo frames (MTU 9000) are essential — with a 1500-byte inner frame,
the outer packet becomes ~1590 bytes, which wouldn't fit in a standard 1500-
byte MTU link.

**Why UDP?** VXLAN uses UDP for two reasons: (1) UDP passes through firewalls
and NAT more easily than raw IP protocols, and (2) the UDP source port is
derived from a hash of the inner frame's headers, which helps routers perform
**ECMP (Equal-Cost Multi-Path)** load balancing. If you have multiple paths
between two VTEPs, different flows get different UDP source ports and are
distributed across paths.

## 5.2 VXLAN Components

**VNI (VXLAN Network Identifier):** A 24-bit ID (0–16,777,215) that identifies
which virtual L2 network a frame belongs to. Think of it as a VLAN on steroids
— instead of 4,094 VLANs, you get over 16 million VNIs. This lab uses
**VNI 330** for the machine network. All PEs agree on this VNI — a frame
tagged with VNI 330 on PE1 will be recognized as the same virtual network on
PE2.

**VTEP (VXLAN Tunnel Endpoint):** The device that encapsulates and
decapsulates VXLAN frames. A VTEP has an IP address that serves as the tunnel
endpoint — this is where VXLAN packets are sourced from and delivered to. Each
PE's l2-vxlan namespace has a VTEP:


| PE  | VTEP IP      | Interface    |
| --- | ------------ | ------------ |
| PE1 | 10.55.96.150 | ve-l2-pe-cmn |
| PE2 | 10.55.96.154 | ve-l2-pe-cmn |
| PE3 | 10.55.96.158 | ve-l2-pe-cmn |


These VTEP IPs are in the CMN VRF subnets. They're reachable across the fabric
because BGP L3VPN distributes the CMN routes between PEs, and SRv6 provides
the transport. This is the critical dependency chain.

## 5.3 The VLAN-Aware Bridge

Instead of one VXLAN device per VNI (which doesn't scale when you have
hundreds of VNIs), this architecture uses a **VLAN-aware bridge** (`br0`) that
maps VLANs to VNIs:

```
VLAN ID 30  ←→  VNI 330
```

**How the bridge works:**

A Linux bridge is a software switch. It operates at Layer 2, learning MAC
addresses from incoming frames and forwarding frames based on its **FDB
(Forwarding Database)**. The bridge has multiple "ports" (interfaces), and it
switches frames between them.

In this architecture, `br0` has these ports:


| Port            | Role                                                          |
| --------------- | ------------------------------------------------------------- |
| `vxlan0`        | VXLAN tunnel interface — carries traffic to/from remote VTEPs |
| `ve-l2-br-home` | The host's machine network connection                         |


When a frame arrives on `ve-l2-br-home` (from the host), the bridge looks up
the destination MAC in its FDB. If the MAC is behind `vxlan0` (i.e., on a
remote PE), the bridge sends the frame out `vxlan0`, which VXLAN-encapsulates
it and sends it to the appropriate remote VTEP.

**Why VLAN-aware?** A VLAN-aware bridge can handle multiple VNIs through a
single bridge using VLAN-to-VNI mapping. Each port is assigned a VLAN, and the
bridge maps VLANs to VNIs automatically. This is more efficient than creating
a separate bridge for each VNI. The production system could add more VLANs/VNIs
without creating additional bridges.

The bridge performs three key functions:

1. **MAC learning** — tracks which MAC addresses are on which port
2. **VLAN filtering** — only forwards frames with the correct VLAN tag
3. **BUM handling** — floods Broadcast, Unknown unicast, and Multicast
  traffic to all VTEPs (since the destination is unknown or "everyone")

**The anycast gateway** (`vlan-30-ednagw`) is a macvlan interface on vlan30
with a **shared MAC and IP** across all PEs:

```
IP:  10.55.99.1/26
MAC: aa:dd:cc:00:ff:ee   ← same on PE1, PE2, and PE3
```

**Why anycast?** In traditional L2 networks, the default gateway is a single
router. If that router fails, every host on the subnet loses connectivity
until a failover protocol (like VRRP) kicks in — which can take seconds.

With an anycast gateway, every PE presents the **exact same** gateway
(same IP, same MAC). When host PE1 (10.55.99.7) sends a packet to its
gateway 10.55.99.1, the ARP response comes from the local PE1's bridge — no
remote traffic needed. If PE1 goes down, hosts on PE2 and PE3 are unaffected
because their gateway is local too. There's no failover because there's no
single point of failure.

## 5.4 Why EVPN?

Traditional VXLAN uses **flood-and-learn**: when the bridge doesn't know where
a MAC address is, it floods the frame to ALL remote VTEPs. Every VTEP receives
it, and the one that has the destination host responds. The bridge then learns
the MAC → VTEP mapping from the response.

This approach has serious problems at scale:

1. **Excessive flooding** — Every unknown MAC triggers a flood to every VTEP.
  In a fabric with 100 VTEPs, that's 99 copies of every unknown frame.
2. **No ARP suppression** — ARP requests are broadcast, so every ARP goes to
  every VTEP, even though only one VTEP can answer.
3. **Slow convergence** — When a host moves between PEs, the old VTEP's FDB
  entry must time out before traffic flows correctly.
4. **No control** — There's no way to implement policy, filtering, or
  prioritization on which MACs are advertised where.

**BGP EVPN** (Ethernet VPN) replaces flood-and-learn with a **control plane**.
Instead of flooding to discover MACs, EVPN uses BGP to distribute MAC/IP
information between VTEPs:

1. When PE1's bridge learns MAC `aa:bb:cc:11:22:33` on `ve-l2-br-home`, FRR's
  zebra daemon picks it up and advertises it via BGP EVPN as a **Type-2 route**
   (MAC/IP advertisement).
2. PE2 and PE3 receive this Type-2 route and install it in their bridge's FDB:
  "MAC `aa:bb:cc:11:22:33` is behind VTEP `10.55.96.150`."
3. When PE2 needs to send a frame to that MAC, it already knows which VTEP to
  use — no flooding required. The frame goes directly to PE1's VTEP.

**EVPN route types in this lab:**


| Type | Name                             | Purpose                                                                                                                                        | Example                            |
| ---- | -------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------- |
| 2    | MAC/IP Advertisement             | Advertises a learned MAC (and optionally its IP) — this is how VTEPs learn where MACs are without flooding                                     | `[2]:[0]:[48]:[0e:7e:db:b9:ee:46]` |
| 3    | Inclusive Multicast Ethernet Tag | Registers a VTEP for BUM (Broadcast, Unknown unicast, Multicast) traffic — this replaces the manual configuration of remote VTEPs for flooding | `[3]:[0]:[32]:[10.55.96.150]`      |


**Type-2 in detail:** When PE1 advertises a Type-2 route, it includes:

- The MAC address that was learned
- Optionally, the IP address associated with that MAC (from ARP/ND snooping)
- The VTEP IP of the advertising PE
- The VNI
- The Route Target (to ensure only matching VNIs import the route)

**Type-3 in detail:** When a PE joins a VNI, it advertises a Type-3 route that
says "I'm a VTEP for VNI 330 at IP 10.55.96.150." Other PEs install this as a
BUM flooding entry — when they need to send a broadcast or unknown unicast
frame, they send it to all VTEPs that advertised Type-3 routes for that VNI.

## 5.5 EVPN Route Reflection

The EVPN BGP (AS 65501) has its own route-reflector topology, **separate from
the L3VPN BGP (AS 65500)**:

```
 spine knows nothing about EVPN
 (different AS, no EVPN sessions)

       PE1 (master-0)
       Route Reflector
       cluster-id 2.2.2.2
        ╱            ╲
       ╱              ╲        iBGP (AS 65501)
     PE2              PE3      over VTEP IPs
     client           client
```

**Why a separate AS?** The L3VPN BGP (AS 65500) runs between pe-router
namespaces and uses IS-IS loopback addresses. The EVPN BGP (AS 65501) runs
between l2-vxlan namespaces and uses VTEP IP addresses. They are completely
separate BGP instances running in separate namespaces with separate FRR
processes. Using different AS numbers makes this explicit and avoids any
accidental session formation.

PE1 reflects EVPN routes between PE2 and PE3. PE2 and PE3 only peer with PE1.
The BGP sessions run over the VTEP IPs (`10.55.96.{150,154,158}`), which are
reachable through the CMN VRF → SRv6 → fabric path. This is why L3VPN must be
fully functional before EVPN can work.

**Why PE1 as EVPN RR (not the spine)?** The spine is the L3VPN RR because it's in the same
AS (65500) and has IS-IS adjacencies to all PEs. But the spine has NO VTEP, NO
l2-vxlan namespace, and NO EVPN BGP instance. It's a pure underlay/L3VPN
device. The EVPN RR must be a PE with an l2-vxlan namespace, so PE1 (the
first PE, designated "master-0") takes this role.

## 5.6 VTEP Reachability — The Critical Chain

For EVPN and VXLAN to work, VTEPs must be able to reach each other. But VTEPs
are in the l2-vxlan namespace, not directly on the fabric. The reachability
chain traverses multiple layers:

```
PE1 l2-vxlan (10.55.96.150)
    │
    │ Step 1: l2-vxlan has a default route → 10.55.96.149
    │ (via ve-l2-pe-cmn → ve-pe-cmn-l2 veth pair)
    ▼
PE1 pe-router CMN VRF
    │
    │ Step 2: CMN VRF has a BGP VPN route:
    │   10.55.96.152/30 → seg6 fd00:30:17:3::
    │   (learned from PE2 via the spine's route reflection)
    ▼
PE1 pe-router default table
    │
    │ Step 3: Default table has an IS-IS route:
    │   fd00:30:17::/48 → via eth1 (link-local next-hop)
    │   SRv6 encapsulate with dst = fd00:30:17:3::
    ▼
spine (transit, normal IPv6 forwarding)
    │
    │ Step 4: IS-IS route: fd00:30:17::/48 → via eth2
    │ (spine doesn't know or care about SRv6 semantics)
    ▼
PE2 pe-router default table
    │
    │ Step 5: SRv6 decapsulate
    │   fd00:30:17:3:: = End.DT4 → CMN VRF (table 4)
    ▼
PE2 pe-router CMN VRF
    │
    │ Step 6: Connected route: 10.55.96.152/30 → ve-pe-cmn-l2
    ▼
PE2 l2-vxlan (10.55.96.154)
```

Count the layers: l2-vxlan → veth → CMN VRF → SRv6 encap → underlay routing →
transit → SRv6 decap → CMN VRF → veth → l2-vxlan. Six transitions, three
protocol layers (IP routing, SRv6, BGP VPN), two namespace crossings.

This is why L3VPN must be working BEFORE EVPN can establish sessions. If any
step in this chain is broken, VTEPs can't reach each other, EVPN sessions
can't form, and VXLAN tunnels have no remote endpoints.

## 5.7 VXLAN + EVPN's Role in This Solution

VXLAN + EVPN is the **top of the stack** — it provides the L2 connectivity
that the machine network needs:

- **VXLAN** provides the data plane: it encapsulates Ethernet frames and
tunnels them between VTEPs over the L3 network.
- **EVPN** provides the control plane: it distributes MAC/IP information
between VTEPs so they know where to send frames without flooding.
- **Together**, they create a virtual Ethernet switch spanning all three PEs.
Any host on the machine network (`10.55.99.0/26`) can ARP for and
communicate with any other host, regardless of which PE it's on.

**The complete protocol stack and each protocol's role:**


| Layer                   | Protocol                    | Role                                              |
| ----------------------- | --------------------------- | ------------------------------------------------- |
| Physical                | Ethernet + fiber            | Moves bits between PE and spine                   |
| Underlay routing        | IS-IS                       | Provides IPv6 reachability between all nodes      |
| Transport encapsulation | SRv6                        | Wraps VRF traffic in IPv6 for fabric transit      |
| VRF route distribution  | BGP L3VPN                   | Distributes VRF routes (including VTEP subnets)   |
| L2 control plane        | BGP EVPN                    | Distributes MAC/IP info between VTEPs             |
| L2 data plane           | VXLAN                       | Tunnels Ethernet frames between VTEPs             |
| Application             | Machine network / workloads | The actual compute nodes that use the L2 connectivity |


Each layer depends on all the layers below it. Remove any layer and everything
above it breaks.

## 5.8 Lab Walkthrough: Verify VXLAN + EVPN

```bash
# SSH into PE1
ssh clab@clab-srv6-lab-pe1

# Check EVPN session status:
sudo podman exec frr-l2-vxlan vtysh -c 'show bgp l2vpn evpn summary'
# Both PE2 (10.55.96.154) and PE3 (10.55.96.158) should show a prefix count

# View EVPN routes:
sudo podman exec frr-l2-vxlan vtysh -c 'show bgp l2vpn evpn'

# Check VNI status:
sudo podman exec frr-l2-vxlan vtysh -c 'show evpn vni'
# VNI 330 should show MACs, ARPs, and 2 remote VTEPs

# Check the bridge FDB (Linux kernel):
sudo ip netns exec l2-vxlan bridge fdb show dev vxlan0
```

**Reading the EVPN route output:**

```
Route Distinguisher: 1.1.1.7:2
*>i[2]:[0]:[48]:[0e:7e:db:b9:ee:46]
                    10.55.96.154
                                       0 65501 i
                    RT:65501:330 ET:8
```

Decoding the route type `[2]:[0]:[48]:[0e:7e:db:b9:ee:46]`:

- `[2]` — Type-2 route (MAC/IP advertisement)
- `[0]` — Ethernet Segment Identifier = 0 (single-homed, no EVPN multi-homing)
- `[48]` — MAC address length in bits (48 bits = 6 bytes = standard MAC)
- `[0e:7e:db:b9:ee:46]` — the actual MAC address being advertised
- `10.55.96.154` — the VTEP behind which this MAC sits (PE2)
- `RT:65501:330` — Route Target (ensures only VNI 330 imports this)
- `ET:8` — Ethernet Tag (used for VLAN-aware services)

**Reading the FDB output:**

```
00:00:00:00:00:00 dst 10.55.96.154 self permanent
```

This is a **BUM flooding entry** — the all-zeros MAC is a wildcard meaning
"all broadcast/unknown unicast/multicast frames." They're sent to PE2's VTEP
(10.55.96.154). Created from the EVPN Type-3 route. `permanent` means it
won't age out — it persists as long as the EVPN session is up.

```
0e:7e:db:b9:ee:46 dst 10.55.96.154 self extern_learn
```

This is a **specific MAC entry** — frames destined for this MAC go directly to
PE2's VTEP. Created from the EVPN Type-2 route. `extern_learn` means it was
learned from the EVPN control plane (BGP), not from data-plane flooding. This
is the key benefit of EVPN: the bridge knows exactly where each MAC is without
ever flooding a single frame.

## 5.9 Dive Deeper: VXLAN and EVPN

- **RFC 7348** — "Virtual eXtensible Local Area Network (VXLAN)" — the core
VXLAN specification
- **RFC 7432** — "BGP MPLS-Based Ethernet VPN" — the EVPN specification
(originally MPLS-based but the BGP concepts are identical for VXLAN)
- **RFC 8365** — "A Network Virtualization Overlay Solution Using EVPN" — how
EVPN is used with VXLAN specifically
- **RFC 9135** — "Integrated Routing and Bridging in EVPN" — IRB (bridging +
routing) in EVPN, covers anycast gateway concepts
- **RFC 9136** — "IP Prefix Advertisement in EVPN" — Type-5 routes for IP
prefix advertisement
- **"EVPN in the Data Center" by Lukas Krattiger et al. (Cisco Press)** —
practical EVPN deployment guide
- **FRR EVPN documentation** — [https://docs.frrouting.org/en/latest/evpn.html](https://docs.frrouting.org/en/latest/evpn.html)
- **Cumulus Networks EVPN guide** — excellent practical resource for Linux-based
EVPN (Cumulus uses the same FRR stack)


---

[← Part 4: BGP & L3VPN](04-bgp-l3vpn.md) | [Back to Index](../LEARNING.md) | [Part 6: Packet Walk →](06-packet-walk.md)
