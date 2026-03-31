# Part 1 — Foundations

Before touching any routing protocol, you need to understand the Linux kernel
features that form the physical plumbing of this architecture. But first, a
quick refresher on some networking fundamentals that underpin everything else.

## 1.0 Networking Primer

If you're comfortable with Layer 2 vs Layer 3, broadcast domains, and how
routing tables work, skip to [1.1](#11-network-namespaces). Otherwise, read on.

### Layer 2 (L2) vs Layer 3 (L3)

Networking is built in layers. The two most important for this guide:

- **Layer 2 (Data Link)** operates on **MAC addresses** and **Ethernet
frames**. L2 devices (switches) forward frames based on a MAC address table.
All devices that can reach each other at L2 without going through a router
are in the same **broadcast domain**. When a device sends an ARP request
("who has 10.0.0.5?"), every device in the broadcast domain hears it.
- **Layer 3 (Network)** operates on **IP addresses** and **packets**. L3
devices (routers) forward packets by consulting a **routing table** that
maps destination IP prefixes to next-hops. A router sits between broadcast
domains and moves packets from one to another.

**Why this matters here:** This architecture needs some nodes to share a Layer 2
broadcast domain (so they can ARP for each other), even though they're on
physically separate servers connected through routers. That's the entire reason
VXLAN exists — it emulates L2 connectivity over an L3 routed network.

### What is a Routing Table?

A routing table is a list of rules the kernel consults when it needs to send a
packet. Each rule says: "for destination prefix X, send the packet to next-hop
Y via interface Z." The kernel picks the **most specific** matching rule
(longest prefix match).

```
Destination        Next-hop         Interface
10.55.96.0/30      directly connected   ve-ovn-pe-cmn
10.55.99.0/26      10.55.96.150     ve-pe-cmn-l2
0.0.0.0/0          192.168.1.1      eth0          (default route)
```

When you type `ip route add 10.0.0.0/8 via 192.168.1.1`, you're adding a
**static route** — a manual entry. Dynamic routing protocols (IS-IS, BGP) do
the same thing automatically: they discover neighbors, exchange reachability
information, and install routes into the kernel's routing table.

### What Does "Forwarding" Mean?

By default, a Linux machine receives packets destined for its own IPs and
discards everything else. When you enable **IP forwarding** (`sysctl net.ipv4.conf.all.forwarding=1`), you tell the kernel: "If a packet arrives
that isn't for me, look up the destination in the routing table and send it
toward the next-hop." This turns a Linux box into a router.

### What is a Broadcast Domain?

A broadcast domain is the set of devices that all receive a Layer 2 broadcast
frame. In a traditional network, a broadcast domain = a VLAN = a subnet. If
you send a broadcast on VLAN 30, every port on every switch that carries
VLAN 30 receives it.

In a data center, servers may be on different physical switches, but you want
them in the same broadcast domain. VXLAN solves this by tunneling L2 frames
over the L3 network — making physically separate switches act as one.

## 1.1 Network Namespaces

A network namespace is an isolated copy of the entire Linux networking stack.
Each namespace has its own:

- **Interfaces** (lo, eth0, veth pairs, etc.) — completely separate sets
- **Routing tables** — independent forwarding decisions
- **Firewall rules** (iptables/nftables) — separate filter chains
- **Sysctl settings** (forwarding, rp_filter, etc.) — independently tunable
- **ARP/neighbor tables** — separate MAC-to-IP mappings
- **Socket bindings** — a process in namespace A cannot see sockets in namespace B

A process running in one namespace **cannot see or interact with** interfaces
in another namespace. This is the same isolation primitive that containers use —
when you run `docker run --network=none`, Docker creates a new network
namespace for that container.

Think of it this way: if you open two laptops side by side, each has its own
wifi adapter, its own IP address, its own routing table, and its own firewall.
Network namespaces give you the same isolation, but on a single machine with
zero hardware. Each namespace behaves as if it were a completely separate
computer from a networking perspective.

**Why this architecture uses namespaces:**

Each physical server runs three logically separate network
functions that would conflict with each other if they shared the same routing
table:


| Namespace   | Role                                          | Analogy                               |
| ----------- | --------------------------------------------- | ------------------------------------- |
| `pe-router` | Provider Edge router — runs IS-IS, BGP, SRv6  | A Cisco/Juniper PE in an SP network   |
| `l2-vxlan`  | Layer 2 gateway — runs VXLAN bridge, BGP EVPN | A VTEP / leaf switch                  |
| `default`   | The host — runs compute workloads             | A customer device plugged into the PE |


Without namespaces, all three would share one routing table. Imagine the
pe-router has an IS-IS route saying "to reach 10.55.96.154, send via eth1 with
SRv6 encapsulation," while the host's default namespace just wants a simple
default route to the internet. If they shared a routing table, these entries
would interfere with each other. A packet from the host might accidentally
follow the PE's SRv6 route, or the PE's BGP traffic might follow the host's
default route. Namespaces give each function its own isolated view of the
network, so they can have completely independent routing decisions.

**Lab commands — explore namespaces:**

```bash
# SSH into a PE VM
ssh clab@clab-srv6-lab-pe1   # password: clab@123

# List all namespaces
ip netns list

# Run a command inside a specific namespace
sudo ip netns exec pe-router ip link show
sudo ip netns exec l2-vxlan ip addr show

# Compare routing tables — they're completely independent:
ip route show                                      # default namespace
sudo ip netns exec pe-router ip route show         # pe-router namespace
sudo ip netns exec l2-vxlan ip route show          # l2-vxlan namespace
```

## 1.2 Veth Pairs (Virtual Ethernet)

A veth pair is a virtual cable with two ends. Whatever goes into one end comes
out the other — it's a bidirectional pipe. The critical feature: **the two ends
can be in different namespaces**.

This is the ONLY way to connect two network namespaces (short of moving a
physical NIC). Think of it as a crossover Ethernet cable between two computers,
except the "computers" are namespaces on the same machine.

**How a veth pair works in detail:**

1. You create a pair: `ip link add vethA type veth peer name vethB`. This gives
  you two interfaces (`vethA` and `vethB`) in the same namespace.
2. You move one end to another namespace: `ip link set vethB netns pe-router`.
  Now `vethA` is in the default namespace and `vethB` is in pe-router.
3. You assign IP addresses to each end from the same subnet (e.g.,
  `10.55.96.22/30` on vethA and `10.55.96.21/30` on vethB).
4. Traffic sent into `vethA` emerges from `vethB`, and vice versa.

The `/30` subnet is important here. A `/30` gives you exactly 4 IP addresses:
the network address, two usable hosts, and the broadcast. It's the standard
choice for point-to-point links because you only need two addresses (one per
end). Each veth pair in this architecture is a `/30` point-to-point link.

```
  ┌─────────────────────────────┐     ┌─────────────────────────────┐
  │       default namespace     │     │    pe-router namespace      │
  │                             │     │                             │
  │  ve-ovn-pe-cmn ─────────────┼─────┼── ve-pe-cmn-ovn            │
  │  10.55.96.22/30             │     │   10.55.96.21/30            │
  │                             │     │   (VRF: CMN)                │
  └─────────────────────────────┘     └─────────────────────────────┘
```

Each PE has four veth pairs, each serving a different traffic type:


| Pair  | End A (namespace)          | End B (namespace)                     | Purpose                                  |
| ----- | -------------------------- | ------------------------------------- | ---------------------------------------- |
| OVN   | `ve-ovn-pe-cmn` (default)  | `ve-pe-cmn-ovn` (pe-router, CMN VRF)  | CMN infrastructure access from host      |
| Host  | `ve-host-pe-cmn` (default) | `ve-pe-cmn-host` (pe-router, CMN VRF) | Host management traffic to PE            |
| L2-PE | `ve-l2-pe-cmn` (l2-vxlan)  | `ve-pe-cmn-l2` (pe-router, CMN VRF)   | VTEP traffic from VXLAN to PE            |
| Home  | `ve-home-l2-br` (default)  | `ve-l2-br-home` (l2-vxlan, bridge)    | Machine network into VXLAN bridge        |

**What the "ovn" veth does:** The `ve-ovn-pe-cmn` / `ve-pe-cmn-ovn` pair gives
the default namespace a direct routed path to the CMN VRF's infrastructure
address space. A static route (`10.55.96.0/24 via 10.55.96.21 dev
ve-ovn-pe-cmn`) gives the host L3 access to CMN subnets — VTEP IPs, pe-router
CMN interfaces, and point-to-point veth endpoints — without hairpinning through
the VXLAN bridge. The "ovn" in the name is a legacy artifact; in this lab it's
simply a routed link to the CMN VRF.

**Naming convention:** The names encode the path. `ve-ovn-pe-cmn` means "veth
end, from the default namespace, going to PE-router, CMN VRF." `ve-pe-cmn-ovn`
is the other end: "from PE-router CMN side, going toward the default
namespace." Reading both ends tells you where the cable runs.

**How they're created (from the setup script):**

```bash
# Create a veth pair in the default namespace
ip link add ve-ovn-pe-cmn mtu 8900 type veth peer name ve-pe-cmn-ovn

# Move one end into the pe-router namespace
ip link set ve-pe-cmn-ovn netns pe-router

# Now ve-ovn-pe-cmn is in default, ve-pe-cmn-ovn is in pe-router
# They act as a point-to-point link between the two namespaces
```

**What is MTU?** MTU (Maximum Transmission Unit) is the largest packet size an
interface will send without fragmenting. Standard Ethernet is 1500 bytes. The
veth pairs use MTU 8900 because traffic flowing through them may already be
VXLAN-encapsulated (adding ~50 bytes of overhead) and then gets SRv6-
encapsulated (adding another ~40 bytes). If the original packet is 1500 bytes,
the fully encapsulated version is ~1590 bytes. With MTU 8900 on the veths
(and 9000 on the fabric link), there's plenty of room for encapsulation
headers without fragmentation. Fragmentation is bad for performance because the
receiving end must reassemble fragments, which consumes CPU and memory.

**Lab commands — inspect veth pairs:**

```bash
# See all interfaces in the default namespace
ip -br link show

# See veth pairs in pe-router (notice they're VRF slaves)
sudo ip netns exec pe-router ip -br link show

# Verify connectivity across a veth pair
sudo ip netns exec pe-router ip addr show ve-pe-cmn-ovn
# Shows 10.55.96.21/30

ip addr show ve-ovn-pe-cmn
# Shows 10.55.96.22/30

# Ping across the veth (works without any routing protocol!)
ping -c 1 -I ve-ovn-pe-cmn 10.55.96.21
```

## 1.3 VRFs (Virtual Routing and Forwarding)

A VRF is a separate routing table instance within a single namespace. While
namespaces isolate the ENTIRE networking stack (interfaces, routing, firewall,
sockets), VRFs only isolate routing — interfaces in different VRFs share the
same kernel but have different forwarding decisions.

**Real-world analogy:** Imagine a post office that serves two different
countries. Each country has its own address system — "123 Main Street" might
exist in both countries but refer to different places. The post office keeps
two separate address books (routing tables). When a letter arrives, the post
office first checks which country it's for (which VRF), then looks up the
address in that country's book. VRFs work the same way: a router can carry
traffic for multiple separate networks, each with its own routing table, even
if they use overlapping IP addresses.

In service provider terminology: VRFs let one router carry traffic for multiple
customers without their routes mixing. Each customer gets their own VRF, and
packets are always forwarded using only that customer's routing table.

**This architecture uses two VRFs in the pe-router namespace:**


| VRF                              | Table ID | Purpose                                       |
| -------------------------------- | -------- | --------------------------------------------- |
| CMN (Cluster Management Network) | 4        | Carries VTEP traffic, CMN infrastructure access, host management |
| EDN (External Data Network)      | 2        | External-facing services (placeholder in lab) |


The underlay (IS-IS, BGP loopback sessions) lives in the **default** table
(table 254) of the pe-router namespace — NOT in any VRF. VRFs are only for
customer/service traffic. This separation is fundamental: the underlay provides
transport between PEs, and VRFs carry the actual service traffic that rides on
top of that transport.

### What is the CMN VRF? (Cluster Management Network)

The CMN VRF is the **backbone of the entire overlay**. It carries all the
internal infrastructure traffic that makes the cluster work:

- **VTEP-to-VTEP traffic:** When a VXLAN-encapsulated frame needs to travel
from PE1's VTEP (10.55.96.150) to PE2's VTEP (10.55.96.154), that packet
is routed through the CMN VRF. The CMN VRF knows (via BGP L3VPN) that PE2's
VTEP subnet (10.55.96.152/30) is reachable via SRv6 SID `fd00:30:17:3::`.
Without CMN, the VXLAN tunnel has no path to the remote VTEP.
- **EVPN BGP sessions:** The FRR instances in the l2-vxlan namespaces peer with
each other over their VTEP IPs. These BGP sessions carry EVPN MAC/IP
advertisements. The VTEP IPs live in the CMN VRF, so these BGP sessions
ride on CMN.
- **Host management traffic:** The veth pairs connecting the default namespace
(where compute workloads run) to the pe-router namespace are enslaved to the CMN VRF.
When the host needs to reach management services on other PEs, that traffic
goes through CMN.
- **CMN infrastructure access:** The "ovn" veth pair (`ve-ovn-pe-cmn` /
`ve-pe-cmn-ovn`) connecting the default namespace to pe-router also lands in
CMN. A static route (`10.55.96.0/24 via 10.55.96.21`) gives the host direct
L3 access to CMN infrastructure addresses — VTEP IPs, pe-router CMN
interfaces, and veth subnets on remote PEs — without hairpinning through the
VXLAN bridge.

**Why CMN exists as a VRF (rather than using the default table):** The default
routing table in pe-router is reserved for the underlay — IS-IS routes, BGP
loopback addresses, and SRv6 locator prefixes. If you mixed CMN traffic into
the default table, CMN subnet routes would interact with IS-IS routes, creating
confusion and potential routing loops. By putting CMN traffic in its own VRF
(table 4), the CMN routes are completely invisible to IS-IS, and IS-IS routes
are invisible to CMN. They only interact through SRv6 encapsulation — CMN
asks SRv6 to deliver a packet to a remote PE, and SRv6 uses the underlay
(default table) to route the encapsulated packet.

**What's in the CMN routing table on PE1:**

```
# ip netns exec pe-router ip route show vrf CMN
10.55.96.148/30 dev ve-pe-cmn-ovn proto kernel  ← local subnet (CMN infrastructure veth)
10.55.96.20/30 dev ve-pe-cmn-host proto kernel   ← local subnet (host veth)
10.55.99.0/26 via 10.55.96.150                   ← machine network → VTEP
10.55.96.152/30 encap seg6 ... fd00:30:17:3::    ← PE2's subnet via SRv6
10.55.96.156/30 encap seg6 ... fd00:30:18:3::    ← PE3's subnet via SRv6
```

The locally-connected subnets are the veth pairs to the default namespace. The
remote subnets (PE2, PE3) are learned via BGP L3VPN and have SRv6
encapsulation instructions attached. The machine network route points to the
VTEP in the l2-vxlan namespace, which gets redistributed into BGP so other
PEs know about it.

### What is the EDN VRF? (External Data Network)

The EDN VRF is designed for **external-facing traffic** — services that need
to reach networks outside the cluster, or that external clients need to reach
from outside.

In a production deployment, EDN would carry:

- **Ingress traffic:** External users accessing applications hosted on the
cluster (e.g., a web application exposed via a load balancer or ingress
controller). The traffic enters through an external-facing interface,
gets routed through EDN to the correct PE, and delivered to the workload.
- **Egress traffic:** Workloads that need to reach external services
(databases, APIs, the internet). The traffic exits through EDN toward an
upstream router or firewall.
- **Load balancer VIPs:** Virtual IPs for load-balanced services that are
reachable from outside the cluster.

**In this lab, EDN is a placeholder.** The VRF is created and BGP is configured
to distribute its routes (with RT `65004:2`), but no workloads actively use
it. The infrastructure is ready — if you added an external-facing interface to
the EDN VRF and configured routes, it would work immediately. EDN is included
in the lab to show the pattern: multiple isolated routing domains sharing the
same PE and the same SRv6 fabric.

**Why EDN is separate from CMN:** Imagine if external traffic and internal
management traffic shared the same routing table. An external route like a
default gateway to the internet (`0.0.0.0/0 via upstream-router`) would affect
ALL traffic in that table — including VTEP traffic and EVPN sessions. A
misconfigured external route could black-hole the entire cluster's internal
communication. By keeping EDN separate:

- CMN routes never see external routes (no accidental traffic leaking)
- EDN can have its own default route without affecting CMN
- Security policies can be applied per-VRF (e.g., firewall rules on EDN's
external interface without touching CMN)
- Each VRF can be independently monitored and debugged

### Why Not Just One VRF?

You might wonder: "If CMN carries all the important internal traffic, why not
put everything in one VRF and skip the complexity?"

The answer is **isolation and safety**. Consider what happens with a single
VRF:

1. An external-facing interface gets added for ingress traffic
2. A default route (`0.0.0.0/0`) is added to reach the internet
3. Now EVERY destination that doesn't match a specific route — including any
  typo'd VTEP address — gets sent toward the internet gateway
4. An attacker on the external network could potentially inject routes that
  hijack internal VTEP traffic
5. A routing loop or black hole in the external path takes down the internal
  overlay too

With separate VRFs, none of these problems exist. CMN is a walled garden —
only the routes it needs are in its table, and nothing external can leak in.

### When Would You Add More VRFs?

VRFs are added whenever you need a new **isolated routing domain** on the same
fabric. Here are real scenarios:

**Scenario 1: Multi-tenant isolation**

You're hosting multiple clusters on the same physical infrastructure.
Cluster A and Cluster B must be completely isolated — different machine
networks, different management, no cross-talk.

```
VRF: CMN-A  (table 10)  ← Cluster A management + VTEPs
VRF: CMN-B  (table 11)  ← Cluster B management + VTEPs
VRF: EDN-A  (table 20)  ← Cluster A external traffic
VRF: EDN-B  (table 21)  ← Cluster B external traffic
```

Each cluster gets its own CMN and EDN. VXLAN bridges and EVPN instances are
duplicated per cluster. Cluster A's VTEPs are in CMN-A, Cluster B's are in
CMN-B. The two clusters share the same physical servers and SRv6 fabric but
are completely invisible to each other at the routing level.

**Scenario 2: Dedicated storage network**

Your cluster needs a high-performance, isolated storage network for Ceph or
similar distributed storage. Storage traffic is latency-sensitive and
bandwidth-heavy — you don't want it competing with management traffic in CMN
or getting affected by external traffic in EDN.

```
VRF: CMN     (table 4)   ← Management + VTEPs (existing)
VRF: EDN     (table 2)   ← External traffic (existing)
VRF: STORAGE (table 5)   ← Dedicated Ceph/NFS traffic
```

Storage nodes get veth pairs into the STORAGE VRF with their own IP subnets.
BGP L3VPN distributes the storage subnets with a unique Route Target (e.g.,
`65004:5`). Storage traffic gets its own SRv6 SID for decapsulation (End.DT4
into table 5). You could even apply different QoS policies to storage traffic
vs management traffic since they're in separate VRFs.

**Scenario 3: Out-of-band management network**

You need a separate management network for BMC/IPMI access, switch management,
or a dedicated monitoring plane that remains reachable even if the cluster's
CNI or overlay is completely broken.

```
VRF: CMN     (table 4)   ← Cluster management (existing)
VRF: EDN     (table 2)   ← External traffic (existing)
VRF: OOB     (table 6)   ← Out-of-band management
```

The OOB VRF connects to a dedicated management switch or VLAN that has direct
access to BMC interfaces. Even if IS-IS crashes, SRv6 breaks, or the VXLAN
overlay is down, the OOB VRF provides a last-resort path to reach the servers'
management controllers.

**The pattern is always the same:**

1. Create a new VRF with a unique table ID
2. Enslave the relevant interfaces to it
3. Add a Route Target for BGP L3VPN distribution
4. FRR auto-allocates an SRv6 SID for the new VRF
5. Remote PEs learn the new VRF's routes and can encapsulate to reach them

Each new VRF costs almost nothing — it's just a routing table in the kernel and
a few BGP attributes. The SRv6 fabric and IS-IS underlay don't change at all.
This is the power of L3VPN: the underlay is built once, and you can layer any
number of isolated routing domains on top of it without touching the physical
network.

**How VRFs work in Linux — step by step:**

```bash
# 1. Create a VRF device (this creates routing table 4)
ip link add CMN type vrf table 4

# 2. Bring it up
ip link set CMN up

# 3. Assign an interface to the VRF
ip link set ve-pe-cmn-ovn master CMN

# 4. Now, traffic arriving on ve-pe-cmn-ovn is looked up in table 4,
#    NOT in the default routing table. The interface is "enslaved" to the VRF.
```

**What "enslaved" means:** When an interface is a "slave" (or "member") of a
VRF, the kernel uses that VRF's routing table for all forwarding decisions
involving that interface. If a packet arrives on `ve-pe-cmn-ovn`, the kernel
looks up the destination in VRF table 4 (CMN), not in the default table (254).
This is how traffic isolation works — even though both tables exist in the same
pe-router namespace, they never interfere with each other.

**Routing table IDs:** Every VRF is backed by a numbered table. Table 254 is
always the default ("main") table. Table 4 is CMN, table 2 is EDN. You can
view any table explicitly: `ip route show table 4` shows CMN's routes. When
you run `ip route show vrf CMN`, it's just a friendlier way to say the same
thing.

**Lab commands — inspect VRFs:**

```bash
# List VRFs
sudo ip netns exec pe-router ip vrf show

# Show routes in a specific VRF
sudo ip netns exec pe-router ip route show vrf CMN

# Show routes in the default table (underlay)
sudo ip netns exec pe-router ip route show table 254

# Show which interfaces belong to which VRF
sudo ip netns exec pe-router ip link show master CMN
```

## 1.4 The Physical NIC Movement

In the real deployment, the server's physical Mellanox NICs (`ens1f0np0`,
`ens1f1np1`) are **moved out of the default namespace** and into pe-router.
This is a critical design decision — it means:

1. The host's default namespace has NO physical network access
2. ALL traffic must flow through pe-router's routing logic
3. pe-router becomes the gateway for everything

This is intentional: by funneling all traffic through the PE router, you get
centralized policy enforcement, consistent encapsulation, and a single point
of control. The host can't accidentally bypass the PE and send unencapsulated
traffic onto the fabric.

In the lab, `eth1` (the vrnetlab data interface connected to the spine) is moved:

```bash
ip link set eth1 netns pe-router
```

After this command, `eth1` disappears from the default namespace (you can't
even see it with `ip link show` in default) and appears in pe-router. IS-IS
runs over this interface to peer with the spine. If you ever need to troubleshoot
the fabric interface, you MUST do it from inside the pe-router namespace:
`sudo ip netns exec pe-router ip link show eth1`.

## 1.5 Putting the Plumbing Together

Here's the complete internal architecture of one PE node after setup. Study
this diagram — it shows every namespace, every veth pair, and every VRF
assignment:

```
┌─ default namespace ──────────────────────────────────────────────────┐
│                                                                      │
│  ve-ovn-pe-cmn ──────────────────────┐                               │
│  10.55.96.22/30                      │                               │
│                                      │                               │
│  ve-host-pe-cmn ─────────────────────┤                               │
│  10.55.97.22/30                      │                               │
│                                      │                               │
│  ve-home-l2-br ──────────────┐      │                               │
│  10.55.99.7/26 (machine net)  │      │                               │
│                               │      │                               │
├───────────────────────────────┼──────┼───────────────────────────────┤
│  l2-vxlan namespace           │      │  pe-router namespace          │
│                               │      │                               │
│  ve-l2-br-home ──┐           │      │  ┌── ve-pe-cmn-ovn  (CMN)    │
│  (bridge slave)   │           │      │  │   10.55.96.21/30           │
│                   │           │      │  │                            │
│  br0 (VXLAN bridge)          │      │  ├── ve-pe-cmn-host (CMN)    │
│    ├─ vxlan0 (VNI 330)       │      │  │   10.55.97.21/30           │
│    ├─ vlan30                  │      │  │                            │
│    │   └─ vlan-30-ednagw     │      │  ├── ve-pe-cmn-l2   (CMN)    │
│    │      10.55.99.1/26      │      │  │   10.55.96.149/30          │
│    │      aa:dd:cc:00:ff:ee  │      │  │                            │
│                               │      │  │   VRF: CMN (table 4)      │
│  ve-l2-pe-cmn ───────────────┼──────┼──┘                            │
│  10.55.96.150/30              │      │                               │
│                               │      │  eth1 (fabric link to spine) │
│  default route → 10.55.96.149│      │  IS-IS + SRv6 underlay       │
│                               │      │                               │
│  FRR: BGP EVPN (AS 65501)    │      │  FRR: IS-IS + BGP (AS 65500) │
└───────────────────────────────┘      └───────────────────────────────┘
```

**Reading this diagram:**

- Lines connecting across the `├───┤` boundary are veth pairs crossing
namespace boundaries
- Interfaces labeled `(CMN)` are enslaved to the CMN VRF — their traffic uses
routing table 4
- `br0` is a Linux bridge (operates at Layer 2, like a virtual switch)
- `vxlan0` is the VXLAN tunnel device (encapsulates/decapsulates frames)
- The l2-vxlan namespace has a default route pointing to pe-router's CMN VRF
— this is how VTEP traffic reaches the fabric


---

[Back to Index](../LEARNING.md) | [Part 2: IS-IS →](02-isis.md)
