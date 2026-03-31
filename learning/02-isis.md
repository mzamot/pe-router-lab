# Part 2 — The Underlay: IS-IS

## 2.0 Dynamic Routing Basics

If you only know static routing, this section bridges the gap. With static
routes, you manually type every route on every router. This works for small
networks but falls apart at scale:

- **No automatic failover.** If a link goes down, static routes don't update.
Traffic black-holes until a human intervenes.
- **Configuration burden.** With N routers, you might need O(N²) static routes.
Add a new subnet? Update every router.
- **No optimization.** Static routes can't adapt to congestion, link costs, or
topology changes.

**Dynamic routing protocols** solve all three by having routers talk to each
other, exchange reachability information, and automatically compute optimal
paths.

### IGP vs EGP

Routing protocols fall into two broad categories:

- **IGP (Interior Gateway Protocol):** Runs WITHIN a single organization's
network (called an Autonomous System, or AS). All routers in the IGP trust
each other and share full topology information. Examples: IS-IS, OSPF, RIP,
EIGRP.
- **EGP (Exterior Gateway Protocol):** Runs BETWEEN organizations. Routers
don't share internal topology — they only advertise "I can reach these
prefixes." The only EGP in use today is **BGP** (Border Gateway Protocol).

In this architecture, **IS-IS is the IGP** (providing reachability within the
fabric) and **BGP is used for overlay services** (distributing VPN and EVPN
routes). BGP is used here in an iBGP configuration (within a single AS), which
is a common service provider pattern.

### Link-State vs Distance-Vector

IGPs come in two fundamental designs:

**Distance-Vector** (e.g., RIP, EIGRP): Each router tells its neighbors "I can
reach network X with cost Y." Routers only know what their neighbors tell
them — they don't see the full topology. Like getting directions from locals:
"the grocery store is 3 blocks north" — you trust them but can't verify.

**Link-State** (e.g., IS-IS, OSPF): Each router broadcasts its **own links**
to EVERY router in the network. Every router builds a complete map of the
entire topology, then independently runs Dijkstra's Shortest Path First (SPF)
algorithm to compute the best path to every destination. Like everyone sharing
a complete map — you can see every road and calculate your own route.

**Why link-state is better for this use case:**

1. **Fast convergence.** When a link fails, the change floods everywhere in
  milliseconds. Every router recomputes paths immediately. Distance-vector
   protocols can take multiple rounds of updates to converge, sometimes
   counting slowly to infinity.
2. **Loop-free by design.** Since every router has the same map and runs the
  same algorithm, they all agree on the best paths. Distance-vector protocols
   need extra mechanisms (split horizon, route poisoning) to prevent loops.
3. **Extensible.** Link-state protocols can carry arbitrary information in their
  database. IS-IS uses this to carry SRv6 locator prefixes — the same
   flooding mechanism that distributes topology information also distributes
   segment routing data.

### How Link-State Routing Works (Simplified)

1. **Discovery.** Each router sends Hello packets on every interface. When two
  routers hear each other's Hellos, they form an **adjacency** (like a
   handshake confirming "we're connected").
2. **Database building.** Each router creates a **Link-State PDU (LSP)** — a
  data structure saying "I am router X, I have links to routers Y and Z with
   costs A and B, and I own these IP prefixes." Each router floods its LSP to
   every other router in the area.
3. **SPF computation.** Once every router has every other router's LSP, they
  each run Dijkstra's algorithm independently. The algorithm takes the
   complete topology graph and computes the shortest path tree from the local
   router to every other router.
4. **Route installation.** The SPF results are converted into routing table
  entries and installed in the kernel. "To reach prefix P, send via interface
   I toward next-hop N with cost C."
5. **Ongoing maintenance.** When a link changes state (up/down), the affected
  router generates a new LSP and floods it. All routers re-run SPF and update
   their routing tables. This happens in milliseconds.

## 2.1 What is an Underlay?

Every overlay network needs an underlay — a simple, fast routing layer that
provides reachability between endpoints. The underlay doesn't know about
customer traffic, VPNs, or VXLAN. Its only job is: **"given a destination IP,
which interface do I send the packet out?"**

Think of the underlay as the highway system and the overlay as the delivery
trucks. The highway system doesn't care what's in the trucks — it just moves
them from city A to city B. The overlay protocols (BGP L3VPN, EVPN) are the
logistics systems that decide what goes in each truck and where it needs to go.

In this architecture:

- **Underlay** = IS-IS running in the pe-router namespace, providing IPv6
reachability between PE loopbacks and SRv6 locators
- **Overlay** = BGP L3VPN (for VRF traffic) and BGP EVPN (for L2 VXLAN),
both riding on top of the underlay

The underlay must be working BEFORE any overlay can function. If the underlay
is broken, overlays have no transport — like trying to run a delivery service
when all the highways are closed.

## 2.2 Why IS-IS?

IS-IS (Intermediate System to Intermediate System) is an IGP (Interior Gateway
Protocol) like OSPF. Both are link-state protocols that build a complete
topology map and run SPF. Service providers overwhelmingly prefer IS-IS over
OSPF for several reasons:

1. **Protocol independence** — IS-IS runs directly on Layer 2 (ISO's
  Connectionless Network Protocol, CLNP). It does NOT run inside IP packets.
   This means IS-IS doesn't depend on IP to function — it can bootstrap an IP
   network from scratch. OSPF runs inside IP (protocol 89), which creates a
   chicken-and-egg problem in some scenarios. IS-IS supports IPv4 and IPv6 as
   equal "topologies" without any protocol changes — it just carries different
   TLVs for each address family.
2. **Stability and extensibility** — IS-IS uses TLVs (Type-Length-Value) for
  all data. Adding a new feature (SRv6 locators, traffic engineering, flex-
   algo) means defining a new TLV type — the core protocol format never
   changes. OSPF uses fixed-format LSAs, and adding features has required
   entirely new LSA types and sometimes new versions of the protocol (OSPFv2
   vs OSPFv3). IS-IS has been essentially the same protocol since the 1990s,
   with capabilities growing via TLV extensions.
3. **Scalability** — IS-IS uses a two-level hierarchy (Level-1 for intra-area,
  Level-2 for inter-area) that scales naturally for large service provider
   networks with thousands of routers. OSPF's area model works but has more
   complex rules about which routers can be area border routers and how routes
   are summarized.
4. **SRv6 integration** — IS-IS advertises SRv6 locator prefixes natively
  using dedicated TLVs (TLV 27 for SRv6 locator, sub-TLVs for SID
   structure). The IETF designed SRv6 with IS-IS as the primary distribution
   mechanism. While OSPF can also carry SRv6 data, IS-IS is the standard
   choice in every major SRv6 deployment.
5. **Industry standard for SP networks** — Nearly every major service provider
  uses IS-IS for their backbone. This
   means better vendor support, more operational tooling, and a larger
   knowledge base.

## 2.3 IS-IS Concepts

### NET (Network Entity Title)

Every IS-IS router has a NET — a unique identifier that encodes its area
membership and system identity. Unlike IP addresses, NETs come from the OSI
addressing world. Don't worry about the OSI details — just understand the
format:

```
49.0000.1000.f006.0016.00
│  │         │         │
│  │         │         └── SEL (always 00 for a router — like saying
│  │         │              "this is a router, not an end system")
│  │         └──────────── System ID (6 bytes, unique per router —
│  │                        like a MAC address for IS-IS)
│  └────────────────────── Area ID (variable length — all routers in the
│                           same area can form Level-1 adjacencies)
└───────────────────────── AFI (49 = private addressing, like 10.x.x.x
                            for IP — used in labs and private networks)
```

For adjacency to form, routers must be in the **same area** (for Level-1) or
any area (for Level-2). All nodes in this lab are Level-1 in area `49.0000`.

**Why all Level-1?** Since this is a small fabric (4 nodes), there's no need
for inter-area routing. Level-1 is simpler: all routers maintain the same
database and have full visibility of the entire topology. In a large SP
network, you'd use Level-2 for the backbone and Level-1 for regional
aggregation — but that adds complexity we don't need here.

### Levels Explained

- **Level-1:** Intra-area routing. All routers in the same area form Level-1
adjacencies and share the same link-state database. Every Level-1 router
knows the full topology within its area. Analogous to OSPF intra-area routes.
- **Level-2:** Inter-area routing. Level-2 routers form a backbone that
connects different areas. A Level-1-2 router sits at the border and
redistributes routes between levels. Analogous to the OSPF backbone (area 0).
- **This lab uses Level-1 only** (`is-type level-1`). All four nodes are in
area `49.0000` and form Level-1 adjacencies. Every node sees every other
node's full topology.

### Adjacency Formation

1. Routers send **IS-IS Hello (IIH)** packets on each IS-IS-enabled interface,
  typically every 10 seconds. IIH packets contain the sender's system ID,
   area ID, and level.
2. When router A hears router B's IIH, it adds B to its Hello packet
  (acknowledging "I see you"). This is the **three-way handshake**: A sends
   IIH → B adds A to its IIH → A sees itself in B's IIH → adjacency is Up.
3. Once the adjacency is Up, the routers exchange their **link-state
  databases** (all the LSPs they have). This is called database
   synchronization.
4. Each router runs **SPF (Dijkstra's algorithm)** to compute shortest paths
  to all destinations, using the synchronized database.
5. SPF results are installed as routes in the kernel routing table by FRR's
  zebra daemon.

### Point-to-Point vs Broadcast

Our interfaces use `isis network point-to-point` because each link connects
exactly two routers (PE ↔ spine). On a point-to-point link:

- No DIS (Designated Intermediate System) election needed. On broadcast
networks (like Ethernet segments with multiple routers), IS-IS elects a DIS
to reduce the number of adjacencies and LSP floods. This election process
adds delay and complexity.
- Faster adjacency formation — the three-way handshake happens directly
between the two routers.
- Simpler SPF calculation — point-to-point links are just edges in the graph.

If a link truly connects only two routers, always use `point-to-point`. It's
simpler, faster, and avoids DIS-related issues.

## 2.4 IS-IS and SRv6: How IS-IS Distributes Segment Routing Data

Section 2.2 mentioned that IS-IS uses TLVs and that this extensibility is key
to SRv6 integration. This section explains the actual mechanism — how IS-IS
carries SRv6 locator information in its link-state database, and why this
matters for the overall architecture.

### TLVs: The IS-IS Extensibility Engine

IS-IS encodes all information inside its LSPs (Link-State PDUs) using **TLVs
(Type-Length-Value)** structures. A TLV is a simple envelope:

```
┌──────────┬──────────┬─────────────────────────────┐
│ Type     │ Length   │ Value                       │
│ (1 byte) │ (1 byte) │ (variable — up to 255 bytes)│
└──────────┴──────────┴─────────────────────────────┘
```

- **Type** identifies what kind of data this is (e.g., 135 = IPv4 reachability,
  236 = IPv6 reachability, 27 = SRv6 locator)
- **Length** says how many bytes the value field contains
- **Value** is the actual data — and it can itself contain **sub-TLVs**
  (nested TLVs within the value field)

When IS-IS was first designed in the 1990s, it carried basic routing information
— interface metrics, IP prefixes, area membership. But because everything is
encoded as TLVs, adding entirely new capabilities is just a matter of defining
a new type number. The IS-IS packet format, flooding mechanism, and SPF
computation don't change at all. This is fundamentally different from OSPF,
which uses fixed-format LSA types — adding a new capability to OSPF often
requires a new LSA type and new processing code.

**The key TLVs in this lab's IS-IS LSPs:**

| TLV Type | Name                        | What It Carries                                      |
| -------- | --------------------------- | ---------------------------------------------------- |
| 236      | IPv6 Reachability           | IPv6 prefixes (loopbacks, connected subnets)         |
| 22       | Extended IS Reachability    | Link adjacencies with metrics (the topology graph)   |
| 27       | SRv6 Locator                | SRv6 locator prefix + SID structure information      |

TLVs 236 and 22 are standard IS-IS — they build the topology map and prefix
table that SPF uses. TLV 27 is the SRv6 extension — it's how IS-IS tells the
network about Segment Routing.

### The SRv6 Locator TLV (Type 27)

When a router is configured with an SRv6 locator, IS-IS advertises it using
**TLV 27** (defined in RFC 9352, "IS-IS Extensions for Segment Routing over
IPv6"). This TLV appears in the router's LSP alongside the standard routing
TLVs.

What TLV 27 contains:

```
TLV 27 — SRv6 Locator
├── Metric: 0                              ← cost to reach this locator (0 = connected)
├── Flags: 0x00                            ← (D-flag would indicate "not this node's locator")
├── Algorithm: 0                           ← SPF algorithm (0 = standard shortest path)
├── Locator prefix: fd00:30:16::/48        ← the SRv6 locator block for this node
│
└── Sub-TLV: SRv6 End SID (type 5)
    ├── SID: fd00:30:16:1::               ← the "End" behavior SID
    ├── Behavior: End with uSID (0x002A)  ← micro-SID endpoint behavior
    └── SID Structure:
        ├── Locator Block Length: 32       ← fd00:0030 (32 bits)
        ├── Locator Node Length: 16        ← 0016 (16 bits)
        ├── Function Length: 16            ← function ID (16 bits)
        └── Argument Length: 0             ← no arguments
```

Breaking this down:

- **Locator prefix** (`fd00:30:16::/48`): The /48 block that this node owns.
  IS-IS floods this prefix to all routers in the area. Every router then has
  a route to `fd00:30:16::/48` via the shortest path toward the advertising
  node. This is exactly how SRv6 packets get routed — the destination address
  falls within a locator prefix, and standard longest-prefix-match forwarding
  delivers the packet to the right node.

- **Algorithm** (0): Which SPF algorithm to use for computing paths to this
  locator. Algorithm 0 is standard SPF (Dijkstra). Flex-Algo (algorithms
  128–255) would allow computing alternative paths with different constraints
  (e.g., "shortest path that avoids link X"), but this lab uses only the
  default algorithm.

- **SRv6 End SID sub-TLV**: Advertises a specific SID and its behavior. The
  `End` behavior (basic endpoint) is advertised through IS-IS. Other SIDs
  like `End.DT4` (VRF decapsulation) are NOT advertised in IS-IS — they're
  distributed by BGP as part of VPN route attributes. This division makes
  sense: IS-IS handles infrastructure-level SIDs (transit forwarding), while
  BGP handles service-level SIDs (VPN delivery).

- **SID Structure**: Tells remote nodes how to parse the 128-bit SID into its
  component parts (block, node, function, argument). This is essential for
  micro-SID (uSID) processing — without knowing the structure, a remote node
  can't determine where the block ends and the function begins.

### What IS-IS Distributes vs What BGP Distributes

This is a common source of confusion. IS-IS and BGP both carry SRv6
information, but they carry **different types**:

| Information               | Distributed By | Why                                                          |
| ------------------------- | -------------- | ------------------------------------------------------------ |
| SRv6 locator prefix       | IS-IS          | Needed by every transit router for basic packet forwarding   |
| SRv6 End SID              | IS-IS          | Infrastructure SID — needed for multi-segment transit paths  |
| SRv6 End.DT4/DT6 SID     | BGP            | Service SID — tied to a VRF, only relevant to VPN endpoints  |
| SRv6 SID structure        | IS-IS          | Needed by all nodes to parse micro-SIDs correctly            |

**The logic**: IS-IS carries information that **every router** in the fabric
needs — locator prefixes for routing and SID structures for parsing. BGP carries
information that **only VPN endpoints** need — which SID to use for reaching a
specific VRF on a specific PE. Transit routers (like the spine) never look at
BGP VPN attributes, but they absolutely need the IS-IS locator routes to
forward SRv6 packets.

### How It Works End-to-End

Here's the sequence when a PE boots and advertises its SRv6 locator:

1. **FRR starts IS-IS** with `segment-routing on` and `segment-routing srv6 /
   locator MAIN` configured under the IS-IS process.

2. **IS-IS generates an LSP** containing the standard TLVs (adjacencies, IPv6
   prefixes) PLUS TLV 27 with the SRv6 locator (`fd00:30:16::/48`) and its
   End SID.

3. **IS-IS floods the LSP** to all neighbors. The spine receives it and floods
   it to the other PEs. Within seconds, every router in the area has PE1's LSP
   in its link-state database.

4. **Every router runs SPF** and computes the shortest path to
   `fd00:30:16::/48`. The result is a routing table entry:
   ```
   fd00:30:16::/48 via fe80::... dev eth1  (metric 10)
   ```
   This is installed in the kernel — now any IPv6 packet destined for an
   address within PE1's locator will be forwarded toward PE1.

5. **BGP separately learns** (from PE1's BGP VPN advertisements) that VRF CMN
   on PE1 uses SID `fd00:30:16:3::`. When a remote PE needs to reach PE1's
   CMN VRF, it encapsulates the packet with destination `fd00:30:16:3::`.
   The kernel matches this against `fd00:30:16::/48` (the IS-IS route) and
   forwards it toward PE1.

6. **When the packet arrives at PE1**, the destination (`fd00:30:16:3::`)
   matches a local SID in the kernel's SRv6 SID table. The kernel performs
   `End.DT4` — decapsulates and delivers to VRF CMN. This SID was programmed
   by FRR's BGP process, not by IS-IS.

The beauty of this division: IS-IS handles **reachability** (how to get packets
to the right node), and BGP handles **service binding** (what to do with
packets once they arrive). Neither protocol needs to understand the other's
job.

### The FRR Configuration

In the lab's FRR config, the IS-IS/SRv6 integration is configured in two
places:

**Inside the IS-IS process** (tells IS-IS to advertise the locator):

```
router isis UNDERLAY
  net 49.0000.0000.0000.0016.00
  topology ipv6-unicast
  segment-routing on
  segment-routing srv6
   locator MAIN
```

`segment-routing on` enables SR extensions in IS-IS. `segment-routing srv6 /
locator MAIN` tells IS-IS to advertise the locator named "MAIN" in its LSPs
using TLV 27.

**In the global segment-routing block** (defines what "MAIN" actually is):

```
segment-routing
 srv6
  encapsulation
   source-address fd00:30:16::
  locators
   locator MAIN
    prefix fd00:30:16::/48 block-len 32 node-len 16
    behavior usid
```

This defines the MAIN locator: prefix `fd00:30:16::/48`, with a 32-bit block
and 16-bit node ID, using micro-SID encoding. The `source-address` is used as
the source IPv6 address in SRv6-encapsulated packets.

IS-IS reads the locator definition from this global block and includes it in
TLV 27. The SID structure information (`block-len 32 node-len 16`) is carried
as a sub-TLV so remote nodes know how to parse addresses within this locator.

### Lab Walkthrough: Verify SRv6 in IS-IS

```bash
# View IS-IS database details — look for TLV 27 (SRv6 Locator):
sudo podman exec frr-pe-router vtysh -c 'show isis database detail'

# In the output, look for lines like:
#   SRv6 Locator:  MT:0 Metric:0 Locator:fd00:30:16::/48 ...
#     SRv6 SID: fd00:30:16:1:: Behavior:End ...
#       SID Structure: LBL:32 LNL:16 FL:16 AL:0

# Verify that remote locators appear as IS-IS routes:
sudo podman exec frr-pe-router vtysh -c 'show ipv6 route isis'

# You should see entries like:
#   I>* fd00:30:1::/48 [115/20] via fe80::..., eth1    ← spine's locator
#   I>* fd00:30:17::/48 [115/20] via fe80::..., eth1   ← PE2's locator
#   I>* fd00:30:18::/48 [115/20] via fe80::..., eth1   ← PE3's locator

# Check the IS-IS SRv6 locator state:
sudo podman exec frr-pe-router vtysh -c 'show segment-routing srv6 locator'
```

### What Happens When a Locator Is Withdrawn

If PE2 goes down (or its IS-IS process crashes), here's the cascade:

1. **The spine stops receiving IS-IS Hellos** from PE2. After the hold timer
   expires (~30 seconds), the spine declares the adjacency down.
2. **The spine generates a new LSP** indicating PE2 is no longer reachable and
   floods it to PE1 and PE3.
3. **All routers re-run SPF.** The route to `fd00:30:17::/48` is withdrawn
   from every routing table. Any SRv6 packet destined for PE2's locator now
   has no route and is dropped.
4. **BGP sessions to PE2 fail** (since they rode on the IS-IS loopback
   reachability). PE2's VPN routes are withdrawn from the BGP table.
5. **CMN VRF routes pointing to PE2** are removed. VTEP reachability to PE2
   is lost. EVPN withdraws PE2's MAC advertisements.
6. **VXLAN traffic to PE2 stops.** BUM flooding entries for PE2's VTEP are
   removed. The remaining PEs (PE1, PE3) continue to communicate normally —
   only PE2 is isolated.

The entire cascade — from IS-IS adjacency loss to VXLAN convergence — typically
completes in under 60 seconds. IS-IS convergence itself is sub-second; the BGP
hold timer (default 180 seconds, often tuned lower) is usually the bottleneck.

## 2.5 IS-IS in This Lab

```
                   ┌──────────┐
                   │  spine   │
                   │ NET: 49.0000.0000.0000.0001.00
                   │ lo: fc00:0:0:1::1/128
                   │ SRv6: fd00:30:1::/48
                   └┬───┬───┬─┘
                   ╱    │    ╲        IS-IS Level-1 p2p
                  ╱     │     ╲
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│     PE1     │  │     PE2     │  │     PE3     │
│ ...0016.00  │  │ ...0017.00  │  │ ...0018.00  │
│ ::16/128    │  │ ::17/128    │  │ ::18/128    │
│ fd00:30:16  │  │ fd00:30:17  │  │ fd00:30:18  │
└─────────────┘  └─────────────┘  └─────────────┘
```

IS-IS advertises two types of IPv6 prefixes for each node:

1. **Loopback** (`fc00:0:0:1::16/128`) — A /128 is a single IPv6 address
  (like a /32 in IPv4). This loopback is used as the BGP update-source,
   meaning BGP sessions are established to this address. Because it's a
   loopback on the router, it's always reachable as long as the router is up
   (unlike a physical interface, which goes down if the cable is pulled). IS-IS
   ensures every router can reach every other router's loopback.
2. **SRv6 locator** (`fd00:30:16::/48`) — A /48 block owned by this node. When
  transit routers see a packet destined for anything within `fd00:30:16::/48`,
   they know to forward it toward PE1. The specific address within the /48
   encodes the **action** the destination PE should perform (e.g., decapsulate
   into VRF CMN). IS-IS advertises this prefix — using TLV 27 as described in
   Section 2.4 — so the entire fabric knows how to route SRv6 packets.

## 2.6 IS-IS's Role in This Solution

IS-IS is the **foundation** that everything else depends on. Without it,
nothing works:

- **BGP L3VPN sessions** are established between loopback addresses. IS-IS
provides the routes that make these loopbacks reachable. No IS-IS = no BGP
sessions = no VPN route exchange.
- **SRv6 data plane** uses the locator prefixes to forward encapsulated
packets. IS-IS advertises these prefixes so transit nodes can route them. No
IS-IS = SRv6 packets have no path = traffic cannot cross the fabric.
- **VTEP reachability** (which EVPN needs) relies on the CMN VRF routes, which
are distributed by BGP L3VPN, which rides on BGP sessions, which require IS-IS
reachability. The dependency chain is:
`IS-IS (transport) → BGP sessions → L3VPN (route distribution) → VRFs know remote subnets → EVPN sessions → VXLAN tunnels`

IS-IS is invisible to the services running on top — the compute workloads in
the default namespace never see an IS-IS packet. But if IS-IS is broken, every
layer above it collapses.

## 2.7 Lab Walkthrough: Verify IS-IS

```bash
# From the spine — check all three PE adjacencies are Up:
sudo podman exec clab-srv6-lab-spine vtysh -c 'show isis neighbor'

# Expected output:
# Area UNDERLAY:
#  System Id           Interface   L  State    Holdtime SNPA
#  dfe696e9efae        eth1        1  Up       28       2020.2020.2020
#  675762f66bf4        eth2        1  Up       28       2020.2020.2020
#  960bbf32736d        eth3        1  Up       27       2020.2020.2020
```

> **Note:** The System IDs look like random hex because they're derived from
> the container's MAC address when FRR auto-generates them. In the real
> deployment, the NETs are explicitly configured with meaningful system IDs.

**Reading the output:**

- **System Id** — the 6-byte unique identifier of the neighbor
- **Interface** — the local interface the adjacency was formed on
- **L** — the IS-IS level (1 = Level-1)
- **State** — `Up` means the adjacency is fully established and the database
is synchronized. `Initializing` means the three-way handshake isn't complete.
- **Holdtime** — seconds until this adjacency is declared dead if no Hello is
received. Typically 30 seconds (3× the 10-second Hello interval).
- **SNPA** — SubNetwork Point of Attachment (essentially the MAC address of the
neighbor's interface)

```bash
# Check the full IS-IS SPF tree:
sudo podman exec clab-srv6-lab-spine vtysh -c 'show isis route level-1'

# Check that IS-IS routes made it into the kernel:
sudo podman exec clab-srv6-lab-spine vtysh -c 'show ipv6 route isis'
```

**What to look for in `show ipv6 route isis`:**

```
I>* fd00:30:16::/48 [115/10] via fe80::..., eth1
```

- `I` = IS-IS learned route (as opposed to `B` for BGP, `C` for connected,
`S` for static)
- `>` = best route (selected by the Routing Information Base after comparing
all sources)
- `*` = installed in kernel FIB (Forwarding Information Base — the kernel is
actually using this route to forward packets)
- `[115/10]` = admin distance 115 (IS-IS's default priority — lower is better;
connected routes are 0, static are 1, OSPF is 110, IS-IS is 115, BGP is 20
for eBGP/200 for iBGP), metric 10 (the IS-IS path cost)
- `via fe80::...` = link-local next-hop. IS-IS on point-to-point links always
uses IPv6 link-local addresses as next-hops (not global addresses). Link-
local addresses are auto-generated from the interface's MAC and are always
present — no configuration needed.

**Troubleshooting:**


| Symptom                          | Check                                              |
| -------------------------------- | -------------------------------------------------- |
| No neighbors                     | `show isis interface` — is the interface in IS-IS? |
| Neighbor stuck in `Initializing` | Area ID mismatch or level mismatch                 |
| Routes missing                   | `show isis database detail` — check LSP contents   |
| Routes in FRR but not in kernel  | `show ipv6 route` — check the FIB flag (`*`)       |


## 2.8 Dive Deeper: IS-IS

- **RFC 1195** — "Use of OSI IS-IS for Routing in TCP/IP and Dual Environments"
— the original specification for running IS-IS with IP
- **RFC 5308** — "Routing IPv6 with IS-IS" — how IS-IS carries IPv6 prefixes
- **RFC 5765** — "Security Threat Analysis for IS-IS"
- **RFC 9352** — "IS-IS Extensions for Segment Routing" — how IS-IS distributes
SRv6 locators and SIDs
- **FRR IS-IS documentation** — [https://docs.frrouting.org/en/latest/isisd.html](https://docs.frrouting.org/en/latest/isisd.html)
- **"IS-IS: Deployment in IP Networks" by Russ White** — the definitive book on
IS-IS operations


---

[← Part 1: Foundations](01-foundations.md) | [Back to Index](../LEARNING.md) | [Part 3: SRv6 →](03-srv6.md)
