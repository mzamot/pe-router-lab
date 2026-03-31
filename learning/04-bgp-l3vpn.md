# Part 4 — BGP and L3VPN

## 4.0 What is BGP? (BGP Basics for Beginners)

### Why Do We Need Another Routing Protocol?

In Part 2, you learned about IS-IS — an Interior Gateway Protocol (IGP) that
discovers the shortest path between routers inside a single network. IS-IS is
fast, efficient, and reliable. So why do we need another routing protocol?

The answer is **scope and purpose**. IS-IS (and OSPF) are designed to work
inside a single administrative domain — one company, one data center, one
service provider's backbone. Every router in the IS-IS domain trusts every other
router. Every router sees the complete topology. This works beautifully inside
one network, but it has two fundamental limitations:

1. **It doesn't work between organizations.** Competing ISPs don't want to
  share their internal router topology with each other. IS-IS requires full
   trust and full topology visibility — that's unacceptable between competing
   ISPs.
2. **It only carries basic IP routes.** IS-IS can advertise "subnet
  `10.0.0.0/24` is reachable via Router A," but it can't carry the rich
   metadata needed for VPN services — things like "this route belongs to
   Customer X's VRF, should be imported by VRFs with Route Target 65500:1,
   and should use SRv6 SID `fd00:30:16:3::` for encapsulation."

**BGP (Border Gateway Protocol)** was invented to solve both of these problems.

### What BGP Is

BGP is a **path-vector routing protocol** that exchanges routing information
between autonomous systems (networks). It's defined in RFC 4271 and is the
single protocol that makes the global internet work — every ISP, cloud
provider, and major enterprise uses BGP to exchange routes with their peers.

Unlike IS-IS or OSPF, which discover the full network topology and compute
shortest paths, BGP takes a fundamentally different approach:

- **No topology sharing.** BGP peers only exchange routes (prefixes), not
topology information. When ISP-A tells ISP-B "I can reach `8.8.8.0/24`,"
it doesn't reveal how many routers or links are inside ISP-A's network.
- **Policy-driven.** BGP doesn't just find the shortest path — it lets
operators apply **policies** to control which routes are accepted, preferred,
or advertised. An ISP might prefer routes through a cheaper peering link even
if a more direct path exists. This policy flexibility is what makes BGP the
"glue" of the internet.
- **TCP-based sessions.** While IS-IS runs directly on Layer 2 (between
directly connected routers), BGP runs over TCP (port 179). This means BGP
peers don't have to be directly connected — they can peer across the network,
as long as TCP connectivity exists. This is critical for iBGP, where route
reflectors may be several hops away.
- **Extensible through address families.** This is the key reason BGP is used
for VPN services. BGP was originally designed for IPv4 internet routes, but
its architecture was extended (RFC 4760, "Multiprotocol Extensions for BGP")
to carry virtually any type of routing information. Each type gets its own
**address family**. This means one BGP session can carry IPv4 routes, IPv6
routes, VPN routes, EVPN MAC advertisements, and more — each in its own
independent channel.

### How a BGP Session Works

When two BGP routers form a session, here's what happens:

1. **TCP connection:** Router A opens a TCP connection to Router B on port 179.
  This requires IP reachability between them (usually provided by the IGP).
2. **OPEN message:** Both routers exchange OPEN messages containing their ASN,
  hold time, router ID, and the address families they support.
3. **UPDATE messages:** Each router sends UPDATE messages containing routes it
  wants to advertise. Each UPDATE includes:
  - **NLRI (Network Layer Reachability Information):** The prefix(es) being
  advertised (e.g., `10.55.96.148/30`)
  - **Path attributes:** Metadata about the route — AS path, next-hop, local
  preference, communities, and (for VPN routes) Route Distinguisher, Route
  Target, and SRv6 SID
4. **KEEPALIVE messages:** Periodic heartbeats (default every 60 seconds) to
  confirm the session is alive. If three consecutive keepalives are missed,
   the session is declared down and all routes from that peer are withdrawn.
5. **Steady state:** After the initial exchange, routers only send UPDATEs when
  something changes — a new route appears, an existing route is withdrawn, or
   a route's attributes change.

### Autonomous Systems (AS)

An AS is a collection of routers under a single administrative control, using
a common routing policy. Each AS has a unique number (ASN). In this lab:

- **AS 65500** — the L3VPN domain (spine + all PEs for underlay BGP)
- **AS 65501** — the EVPN domain (all PE l2-vxlan instances)

ASN 65500–65534 are "private" AS numbers (like 10.x.x.x for IP), used for
internal purposes.

### iBGP vs eBGP

- **eBGP (external BGP):** Sessions between routers in DIFFERENT autonomous
systems. This is how ISPs exchange routes. eBGP peers are typically directly
connected. eBGP routes have admin distance 20 (high priority).
- **iBGP (internal BGP):** Sessions between routers in the SAME autonomous
system. This is how routers within an organization share routing information.
iBGP peers are often NOT directly connected — they peer over loopback
addresses (which the IGP makes reachable). iBGP routes have admin distance
200 (lower priority than eBGP).

**Critical iBGP rule:** A router that learns a route via iBGP CANNOT re-
advertise it to another iBGP peer. This prevents routing loops but creates
a problem: with N routers, every router must peer with every other router
(full mesh = N×(N-1)/2 sessions). For 100 routers, that's 4,950 sessions.
This doesn't scale.

**Route Reflectors** solve this. A route reflector (RR) is allowed to re-
advertise iBGP routes to other iBGP peers. All routers peer with the RR
instead of with each other. N routers → N sessions (instead of N²).

In this lab, **the spine is the L3VPN route reflector** and **PE1 is the EVPN route
reflector**.

### Address Families

BGP was originally designed for IPv4 routes only. Over time, it was extended
to carry many types of routing information through **address families** (AFI)
and **sub-address families** (SAFI):


| Address Family   | What It Carries                   | Used In This Lab?            |
| ---------------- | --------------------------------- | ---------------------------- |
| IPv4 Unicast     | Standard IPv4 routes              | Yes (within VRFs)            |
| IPv6 Unicast     | Standard IPv6 routes              | Yes (IS-IS loopback peering) |
| IPv4 VPN (VPNv4) | VRF routes with RD + RT metadata  | Yes (L3VPN)                  |
| IPv6 VPN (VPNv6) | IPv6 VRF routes with RD + RT      | Yes (L3VPN)                  |
| L2VPN EVPN       | MAC/IP advertisements + VTEP info | Yes (EVPN)                   |


Each address family is independently activated per neighbor. You can have a BGP
session that carries IPv4 VPN routes but NOT IPv6 unicast routes, for example.
This is why the config says `no bgp default ipv4-unicast` — it disables the
automatic activation of IPv4 unicast for every peer, so you can selectively
enable only the families you need.

### Why BGP Is Used in This Architecture

Now that you understand what BGP is and how it works, here's why this
architecture needs it. In this lab, BGP serves two distinct but equally
critical purposes:

1. **L3VPN route distribution (Section 4.2):** BGP carries VRF routes (CMN, EDN
  subnets) between PE routers using the IPv4 VPN address family. Each route
   includes an SRv6 SID telling the remote PE how to encapsulate packets. IS-IS
   can't do this — it has no concept of VRFs, Route Targets, or SRv6 SIDs for
   VPN services.
2. **EVPN MAC/IP advertisement (Part 5):** BGP carries MAC address and VTEP
  information between l2-vxlan namespaces using the L2VPN EVPN address family.
   This is what makes VXLAN scale — instead of flooding to discover MACs, VTEPs
   learn them through BGP.

Neither IS-IS nor any other IGP can carry this kind of metadata. BGP's address
family architecture makes it the only routing protocol that can simultaneously
handle internet routing, VPN routes, and EVPN MAC advertisements — each in
its own independent channel over the same session.

**In summary:**

- **IS-IS** tells routers how to reach each other (underlay connectivity)
- **BGP** tells routers about VPN routes and MAC locations (overlay services)
- IS-IS is the "roads," BGP is the "delivery service" that uses those roads

## 4.1 What is a VPN? (Not What You Think)

Now that you understand BGP — what it is, how sessions work, and how address
families let it carry different types of routing information — we need to talk
about **VPNs** before we go further. But not the kind you're thinking of.

### The Word "VPN" Has Been Hijacked

When most people hear "VPN," they picture a consumer app — NordVPN, Mullvad,
or the corporate WireGuard tunnel you use to access your office network from
home. These are **point-to-point encrypted tunnels** that hide your traffic
from your ISP and make it look like you're browsing from somewhere else.

That's not what VPN means here. Not even close.

In service provider networking, "VPN" has a much older and broader meaning that
predates those consumer apps by decades. A **Virtual Private Network** is any
technology that makes multiple separate sites appear to share a single private
network, even though the traffic between them crosses shared infrastructure
that other customers also use.

The "Virtual" means it doesn't require dedicated physical links between sites.
The "Private" means each customer's traffic is isolated — they can't see each
other's packets. The "Network" means it's a complete routing domain, not just
a single tunnel between two points.

### The Problem: Connecting Sites over Shared Infrastructure

Consider the fundamental problem every service provider faces. You run a
nationwide backbone network — hundreds of routers, thousands of miles of fiber.
You have enterprise customers who need to connect their offices:

- **Acme Corp** has offices in New York, Chicago, and Los Angeles. They want
  all three offices to communicate as if they were on one private LAN.
- **Globex Inc** has offices in the same three cities. They also want their
  offices interconnected.
- Both companies want to use `10.0.0.0/8` internally, because that's their
  existing addressing scheme and they're not going to renumber.

The most obvious solution: run dedicated fiber between each customer's offices.
Acme gets their own links, Globex gets their own links, everybody's happy. But
this is absurdly expensive. You'd need separate physical infrastructure for
every customer, most of it sitting idle most of the time.

The next idea: **share the infrastructure** but keep each customer's traffic
separate. This is the VPN problem, and the industry has been solving it in
progressively better ways for 40 years:

**Generation 1 — Dedicated circuits (1980s):**
Frame Relay and ATM gave each customer dedicated virtual circuits over shared
physical links. The service provider provisioned a virtual circuit between
every pair of customer sites. This worked, but required the provider to
manually configure a circuit for every site-to-site connection. Adding a new
office meant calling the provider and waiting for them to provision new
circuits to every existing office. For a customer with 100 sites, that's
4,950 circuits (N×(N-1)/2). It didn't scale.

**Generation 2 — Tunnels (1990s):**
GRE tunnels and IPsec gave customers the ability to build their own VPNs over
the public internet. Each site creates encrypted tunnels to every other site.
This removed the provider from the equation but pushed all the complexity onto
the customer. Each site needs to maintain tunnels and routing to every other
site. The full-mesh problem returns — it's just moved from the provider to the
customer. Performance is also unpredictable because traffic crosses the public
internet.

**Generation 3 — Provider-provisioned IP VPNs (2000s–present):**
This is the breakthrough: **BGP/MPLS VPNs** (RFC 4364, later extended to
SRv6). The service provider's backbone routers natively understand VPNs. Each
customer gets a **VRF (Virtual Routing and Forwarding instance)** on each PE
router where they connect. The provider's BGP distributes VRF routes between
PEs automatically. When Acme's New York office sends a packet to their LA
office, the ingress PE looks up the destination in Acme's VRF, encapsulates
the packet, and the backbone delivers it to the right egress PE and the right
VRF. No manual circuits, no customer-managed tunnels, no full-mesh scaling
problems.

This third generation is what "VPN" means in this architecture.

### What Makes a Provider VPN Different from a Consumer VPN

| Aspect | Consumer VPN (NordVPN, WireGuard) | Provider VPN (L3VPN) |
| --- | --- | --- |
| **Topology** | Point-to-point tunnel | Any-to-any mesh, automatically |
| **Who manages it** | The end user | The network infrastructure |
| **Routing** | Static (one tunnel, one path) | Dynamic (BGP distributes routes) |
| **Scaling** | Each new site = manually configure a new tunnel | Each new site = add a VRF, BGP does the rest |
| **Isolation method** | Encryption | Separate routing tables (VRFs) + encapsulation |
| **Address overlap** | Not handled | Fully supported (Route Distinguishers) |
| **Traffic path** | Over the public internet | Over the provider's engineered backbone |

The key difference is **the network itself is VPN-aware**. In a consumer VPN,
the network is dumb — it just carries encrypted blobs between two endpoints. In
a provider VPN, the PE routers understand VRF membership, route targets, and
encapsulation. The VPN is a first-class feature of the infrastructure, not an
overlay bolted on top.

### Why This Matters for THIS Architecture

In this lab, the "service provider backbone" is the SRv6 fabric running
between the PE routers. The "customers" are the network namespaces inside each
server — the default namespace (compute workloads), the l2-vxlan namespace (VXLAN
bridge), and the services reachable through each VRF (CMN for management/VTEP
traffic, EDN for external-facing traffic).

The problem is the same one service providers solved: **multiple isolated
traffic domains need to communicate across shared infrastructure.** The CMN
VRF on PE1 needs to reach the CMN VRF on PE2 (so VTEPs can talk to each
other). The EDN VRF on PE1 needs to reach the EDN VRF on PE2 (so external
traffic can flow). But CMN and EDN must stay isolated from each other and from
the IS-IS underlay — just like Acme and Globex must stay isolated from each
other on a service provider's backbone.

This is the VPN problem. The technology that solves it is **L3VPN**.

## 4.2 What is L3VPN? (The Big Picture)

With the VPN problem framed, we can now talk about the specific technology
that solves it: **L3VPN**.

### L2 vs L3 VPNs

VPNs come in two flavors, distinguished by the OSI layer at which they
provide connectivity:

- **Layer 2 VPN (L2VPN):** Connects sites at the Ethernet frame level. Sites
  share a virtual LAN — they can ARP for each other, exchange broadcast
  frames, and see each other's MAC addresses. The provider network forwards
  Ethernet frames between sites. VPLS and EVPN are L2VPN technologies. (EVPN
  is covered in Part 5.)
- **Layer 3 VPN (L3VPN):** Connects sites at the IP routing level. Each site
  has its own subnet, and the provider network routes IP packets between
  them. Sites don't share a broadcast domain — they communicate through
  routed hops. BGP/MPLS VPN (RFC 4364) and its SRv6 successor are L3VPN
  technologies.

The "3" in L3VPN means the VPN operates at Layer 3 — **IP routing**. Each
customer site connects to a PE router, which maintains a VRF with that
customer's routes. The PE routers exchange VRF routes via BGP and encapsulate
traffic across the backbone. The customer sees a routed network where every
site can reach every other site by IP address.

### The Problem L3VPN Solves (Precisely)

L3VPN solves three specific problems simultaneously:

1. **Route isolation:** Multiple customers (or traffic domains) share the same
   physical routers, but their routes must be completely separate. Customer A's
   route to `10.0.0.0/8` must not interfere with Customer B's route to
   `10.0.0.0/8`. VRFs solve this — each customer gets their own routing table.
2. **Automatic route distribution:** When a new site comes online, every other
   site in the same VPN must learn about its subnets. BGP's VPN address
   families automate this — routes are distributed with metadata (Route
   Targets) that tells each PE which VRF should receive them.
3. **Transport across the backbone:** Customer packets must cross the provider's
   backbone without leaking into other VPNs or the global routing table. SRv6
   (or MPLS) encapsulation wraps customer packets in transit-safe headers that
   backbone routers forward without inspecting the contents.

### How L3VPN Works (Conceptually)

The key components:

1. **VRFs (Virtual Routing and Forwarding):** Each customer gets their own
  routing table on each PE (Provider Edge) router. Customer A's routes go into
   VRF-A, Customer B's routes into VRF-B. Even if both use `10.0.0.0/8`, there's
   no conflict because the routes live in separate tables. (You learned about
   VRFs in Part 1 — now you see why they exist.)
2. **BGP with VPN address families:** PE routers use BGP to exchange VRF routes
  with each other. Each route is tagged with a Route Distinguisher (for
   uniqueness) and Route Target (for VRF membership), so the receiving PE knows
   which VRF to put it in. This is how PE1 learns that PE2's CMN VRF has
   subnet `10.55.96.152/30`. (This is the address family mechanism you just
   learned about in Section 4.0.)
3. **Transport encapsulation:** When a PE needs to send a VRF packet across the
  backbone to another PE, it wraps the packet in a transport header. In
   traditional L3VPN this was MPLS labels; in this architecture it's SRv6. The
   backbone routers just forward the encapsulated packet — they don't know or
   care about the VRF contents inside.

### Where L3VPN Is Normally Used

L3VPN is the **bread and butter** of every major service provider network:

- **Enterprise connectivity:** A bank with 500 branches uses L3VPN to connect
all branches over the SP's backbone. Each branch's PE has a VRF for the bank,
and BGP distributes the branch subnets so every branch can reach every other
branch.
- **Mobile backhaul:** Cell towers connect to the core network over an L3VPN.
Each tower's traffic is in a VRF, isolated from other services on the same
transport network.
- **Multi-tenant data centers:** Different tenants get different VRFs, each
with its own isolated routing domain. A tenant's workloads on Rack A can
reach their workloads on Rack B through the VPN, without seeing other
tenants' traffic.
- **Cloud provider networks:** AWS, Azure, and GCP all use L3VPN concepts
internally to isolate customer VPCs (Virtual Private Clouds).

L3VPN has been deployed at massive scale for over 20 years. It's one of the
most proven, battle-tested networking technologies in existence.

### What L3VPN Does in THIS Architecture

In this architecture, L3VPN serves the same purpose it serves in any service
provider network: **distributing VRF routes between PE routers so that every
namespace connected to a VRF can reach its counterpart on remote PEs.**

Each PE has a CMN VRF with several locally connected subnets — the veth pairs
that connect the pe-router namespace to the default namespace and the l2-vxlan
namespace (as you saw in Part 1). Without L3VPN, PE1's CMN VRF only knows
about its own local subnets. It has no idea that PE2 has a VTEP at
`10.55.96.154` or a host management interface at `10.55.97.25`. With L3VPN,
BGP distributes all of these subnets between PEs, and each route includes an
SRv6 SID telling the receiving PE how to encapsulate packets to reach the
remote subnet.

The consumers of L3VPN in this architecture are:

- **VTEP traffic (l2-vxlan namespace):** The VXLAN endpoints on each PE need
  to reach each other to tunnel L2 frames across the fabric. This is the most
  critical dependency — without it, the entire Layer 2 overlay collapses.
- **CMN infrastructure access (default namespace):** The "ovn" veth pair
  gives the host direct L3 access to CMN infrastructure addresses (VTEP IPs,
  pe-router CMN interfaces) via a static route, without hairpinning through
  the VXLAN bridge. L3VPN distributes these subnets so CMN endpoints on
  remote PEs are reachable.
- **Host management traffic (default namespace):** A separate veth pair
  connects the default namespace to the CMN VRF for dedicated host management
  traffic (BMC-related, monitoring, etc.), isolated from the CMN
  infrastructure veth for policy and QoS purposes.
- **EDN traffic (external-facing services):** The EDN VRF distributes routes
  for external-facing services across PEs using the same L3VPN mechanism, just
  with a different Route Target.

The customers in this "VPN" aren't external enterprises — they're the
namespaces within each PE (default namespace, l2-vxlan namespace). The CMN VRF
carries VTEP, CMN infrastructure, and host management traffic. The EDN VRF carries
external-facing traffic. L3VPN keeps both isolated from each other and from the
IS-IS underlay.

**The critical chain for the L2 overlay** (the most visible dependency):

IS-IS + SRv6 provide the transport between PEs. L3VPN distributes the route
knowledge so each PE's CMN VRF knows remote subnets exist and how to
encapsulate toward them. Without L3VPN, the transport is there but the VRF has
no routes — it doesn't know PE2's VTEP subnet exists, so EVPN sessions can't
form, VXLAN tunnels can't build, and the machine network has no path.

## 4.3 Why BGP for VPN?

IS-IS gives us IP reachability between routers. But VRF routes (the customer
networks inside CMN and EDN) need a different distribution mechanism because:

1. **VRF routes are private.** They shouldn't pollute the global routing table.
  If PE1's CMN VRF has a route to `10.55.96.20/30`, that route should NOT
   appear in the underlay routing table. It belongs only in CMN VRFs on other
   PEs. BGP's VPN address families keep these routes in a separate namespace.
2. **Multiple VRFs might use overlapping IPs.** Imagine two customers both using
  `10.0.0.0/8`. In the underlay, there can be only one route to `10.0.0.0/8`.
   But in BGP VPN, each VRF's routes are tagged with a Route Distinguisher
   (RD), making them globally unique: `RD1:10.0.0.0/8` ≠ `RD2:10.0.0.0/8`.
3. **Routes need metadata.** Each VPN route carries additional information: which
  VRF it belongs to (Route Target), how to reach it (SRv6 SID), who
   originated it (router-id). BGP's attribute system carries all of this
   naturally. IS-IS has no concept of VRF membership or SRv6 SIDs for services.
4. **Policy control.** BGP's Route Target import/export mechanism gives fine-
  grained control over which VRFs share routes. A route exported with RT
   `65004:4` (CMN) will only be imported by VRFs that are configured to accept
   that RT. You could have 100 VRFs on a PE and each would only see the routes
   it's supposed to.

## 4.4 Key Concepts

### Route Distinguisher (RD)

The RD makes every VPN prefix globally unique. It's a 64-bit value prepended
to each route when it enters the VPN address family. Format is typically
`router-id:vrf-number`.

```
PE1 CMN VRF advertises:    172.31.250.106:4:10.55.96.20/30
PE2 CMN VRF advertises:    172.31.250.107:4:10.55.96.24/30
```

Even if both PEs had the same subnet, the different RDs would keep them
distinct in BGP's table. The RD is ONLY used for uniqueness — it does NOT
control which VRFs import the route. That's the Route Target's job.

### Route Target (RT)

The RT controls which VRFs import which routes. It's a BGP extended community
attached to each VPN route.

- **Export RT:** Attached when a route leaves a VRF into the VPN address family.
"I belong to this group."
- **Import RT:** Checked when a route is being considered for installation into
a VRF. "I accept routes from this group."

In this lab:

- RT `65004:4` = CMN VRF routes. All PEs export their CMN routes with this RT
and import routes carrying this RT into their CMN VRFs.
- RT `65004:2` = EDN VRF routes. Same pattern for the EDN VRF.

`rt vpn both 65004:4` means "use RT 65004:4 for BOTH export AND import." This
is the common case where a VRF wants to see all other VRFs' routes that share
the same RT.

### Route Reflector (RR)

In iBGP, the full-mesh requirement means every router must peer with every
other router. A route reflector breaks this by allowing one router to re-
advertise iBGP routes. All clients peer only with the RR.

```
Without RR (full mesh — 6 sessions for 4 nodes):
    PE1 ─── PE2
     │ ╲   ╱ │
     │  ╲ ╱  │
     │  ╱ ╲  │
     │ ╱   ╲ │
    PE3 ─── spine

With RR (star — 3 sessions):
         ┌──────┐
         │spine │  Route Reflector
         └┬──┬──┤
          │  │  │
     ┌────┘  │  └────┐
     │       │       │
   ┌─┴─┐  ┌─┴─┐  ┌─┴─┐
   │PE1│  │PE2│  │PE3│  Clients
   └───┘  └───┘  └───┘
```

**How reflection works:**

1. PE1 sends a VPN route to the spine (the RR)
2. The spine "reflects" (re-advertises) the route to PE2 and PE3
3. The spine sets the `originator-id` attribute to PE1's router-id, so PE1 doesn't
  re-import its own route
4. The spine adds its own `cluster-id` to the `cluster-list` attribute, preventing
  reflection loops in multi-RR designs

The RR is a **control plane only** concept. Data plane traffic does NOT flow
through the RR. When PE2 sends a packet to PE1's VRF, it goes directly
PE2 → spine → PE1 via SRv6. The spine happens to be on the path because of the star
topology, but the RR function has nothing to do with it — the spine is just a transit
router in the underlay.

## 4.5 How L3VPN Route Exchange Works

Step by step for CMN VRF on PE1:

1. **Local routes exist:** PE1's CMN VRF has connected routes like
  `10.55.96.20/30` (the CMN infrastructure veth subnet) and `10.55.96.148/30` (the
   l2-vxlan veth subnet). These are the subnets on the veth pairs inside the
   CMN VRF.
2. **Export to VPN:** The config says `redistribute connected` and
  `export vpn`. BGP takes each CMN route and wraps it with VPN metadata:
  - Adds RD `172.31.250.106:4` (making it globally unique)
  - Adds RT `65004:4` (marking it as a CMN route)
  - Attaches the SRv6 SID `fd00:30:16::` (auto-allocated by
  `sid vpn per-vrf export auto` — this tells remote PEs how to reach this
  VRF)
3. **Advertise to RR:** PE1 sends the wrapped VPN route to the spine via the iBGP
  session (which runs over IS-IS loopback addresses).
4. **Reflect:** The spine, acting as route reflector, sends the route to PE2 and PE3
  (because they're configured as `route-reflector-client`). The spine doesn't need
   to understand the SRv6 SID or VRF — it just reflects the BGP update.
5. **Import from VPN:** PE2 receives the route, checks the RT (`65004:4`
  matches its CMN VRF's import policy), and imports it into its CMN VRF. The
   SRv6 SID tells PE2 how to encapsulate packets destined for PE1's CMN.
6. **Install in FIB:** PE2's CMN VRF now has:
  ```
   10.55.96.148/30 via fc00:0:0:1::16, seg6 fd00:30:16:3::
  ```
   "To reach PE1's VTEP subnet, SRv6-encap to `fd00:30:16:3::`."

## 4.6 BGP L3VPN's Role in This Solution

BGP L3VPN is the **brain** that tells each PE how to reach every other PE's
VRF subnets:

- **Without BGP L3VPN:** PE1's CMN VRF only knows about its own connected
subnets. It has no idea that PE2's VTEP is at `10.55.96.154`, or that PE2's
host management subnet is `10.55.97.24/30`, or how to reach either of them.
Every namespace on PE1 — l2-vxlan, default (host) — would be isolated from
its counterparts on PE2 and PE3.
- **With BGP L3VPN:** PE1's CMN VRF learns (via the spine's route reflection)
that `10.55.96.152/30` (PE2's VTEP subnet), `10.55.97.24/30` (PE2's host
management subnet), and every other CMN-connected subnet on PE2 can be
reached by SRv6-encapsulating to `fd00:30:17:3::`. Similarly for PE3's
subnets.
- **Critical for the entire cluster:** L3VPN distributes route knowledge for
all VRF subnets across PEs. IS-IS and SRv6 provide the underlying transport,
but without L3VPN the CMN VRF has no routes to remote subnets — it doesn't
know they exist. The VTEP dependency is the most visible (without L3VPN →
no VRF routes → no path to remote VTEPs → no EVPN → no VXLAN → no L2
machine network), but CMN infrastructure access and host management traffic
also depend on L3VPN for route distribution.

The dependency chain (focusing on the L2 overlay path):

```
IS-IS (underlay reachability)
  → enables BGP L3VPN sessions (over loopback addresses)
    → distributes CMN VRF routes (VTEP, CMN infrastructure, host mgmt subnets)
      → each PE's CMN VRF now has routes + SRv6 SIDs for remote subnets
        ├─ VRF knows remote VTEP subnets → EVPN BGP sessions can form → VXLAN tunnels → L2 machine network
        ├─ VRF knows remote CMN subnets → host can reach CMN infrastructure on other PEs
        └─ VRF knows remote mgmt subnets → host management traffic can reach other PEs
```

## 4.7 The BGP Configuration Explained

```
router bgp 65500
 bgp router-id 172.31.250.106          ← unique 32-bit ID for this BGP speaker
 no bgp default ipv4-unicast           ← don't auto-activate ipv4 unicast for all peers
 bgp default ipv6-unicast              ← auto-activate ipv6 unicast (we peer over IPv6)
 bgp allow-martian-nexthop             ← allow SRv6 addresses (fd00::/8) as next-hops
                                           (normally, BGP rejects "martian" addresses that
                                            aren't globally routable)

 neighbor SPINE peer-group                ← define a template for the spine peer
 neighbor SPINE remote-as 65500           ← iBGP (same AS number = internal BGP)
 neighbor SPINE capability extended-nexthop  ← carry IPv4 VPN routes with IPv6 next-hops
                                             (we peer over IPv6 but the VPN prefixes are IPv4)
 neighbor fc00:0:0:1::1 peer-group SPINE  ← spine's IS-IS loopback as the peering address

 segment-routing srv6
  locator MAIN                         ← use the MAIN locator for auto-allocating VPN SIDs
 sid vpn per-vrf export auto           ← each VRF gets its own automatically allocated SID

 address-family ipv4 vpn
  neighbor SPINE activate                 ← enable VPN route exchange with spine
 exit-address-family

router bgp 65500 vrf CMN              ← BGP config specific to the CMN VRF
 sid vpn per-vrf export auto           ← allocate an SRv6 SID for this VRF

 address-family ipv4 unicast
  redistribute kernel                  ← include kernel-installed routes (e.g., from zebra)
  redistribute connected               ← include connected subnets (veth /30s)
  redistribute static                  ← include static routes (machine network route)
  rd vpn export 172.31.250.106:4       ← Route Distinguisher for uniqueness
  rt vpn both 65004:4                  ← Route Target — import AND export with this tag
  export vpn                           ← push CMN routes into the VPN address family
  import vpn                           ← pull remote CMN routes from the VPN into this VRF
 exit-address-family
```

## 4.8 Lab Walkthrough: Verify BGP L3VPN

```bash
# Check BGP session status on the spine:
sudo podman exec clab-srv6-lab-spine vtysh -c 'show bgp ipv4 vpn summary'
# Look for: State/PfxRcd = 5 (each PE exports ~5 CMN routes)
```

**Reading the summary output:**

```
Neighbor           V  AS  MsgRcvd  MsgSent  InQ OutQ  State/PfxRcd
fc00:0:0:1::16     4 65500     45       48    0    0          5
```

- **Neighbor** — the peer's address (PE1's loopback)
- **V** — BGP version (always 4)
- **AS** — the peer's AS number (65500 = iBGP)
- **State/PfxRcd** — if it shows a number (like 5), the session is Established
and the peer has sent 5 prefixes. If it shows a word like `Active` or
`Connect`, the session isn't up yet.

```bash
# View all VPN routes on the spine (the reflected routes):
sudo podman exec clab-srv6-lab-spine vtysh -c 'show bgp ipv4 vpn'
```

**Reading VPN route output:**

```
Route Distinguisher: 172.31.250.106:4     ← from PE1, VRF CMN
 *>i 10.55.96.148/30  fc00:0:0:1::16     ← the VTEP subnet
   UN=fc00:0:0:1::16 EC{65004:4}         ← underlay next-hop + Route Target
   sid=fd00:30:16:: sid_structure=[32,16,16,0]  ← SRv6 SID + structure
```

- `*>i` — best route (`>`), valid (`*`), learned via iBGP (`i`)
- `10.55.96.148/30` — the VPN prefix (PE1's VTEP veth subnet)
- `fc00:0:0:1::16` — the BGP next-hop (PE1's loopback)
- `EC{65004:4}` — Extended Community = Route Target 65004:4 (CMN)
- `sid=fd00:30:16::` — the SRv6 SID for reaching this prefix
- `sid_structure=[32,16,16,0]` — block 32 bits, node 16 bits, function 16 bits,
argument 0 bits

```bash
# On PE1 — check what the CMN VRF routing table looks like:
sudo podman exec frr-pe-router vtysh -c 'show ip route vrf CMN'

# Key entries to find:
# C>* 10.55.96.148/30  ... connected     ← local VTEP subnet
# B>  10.55.96.152/30  ... seg6 fd00:30:17:3::  ← PE2's VTEP subnet via SRv6
# B>  10.55.96.156/30  ... seg6 fd00:30:18:3::  ← PE3's VTEP subnet via SRv6
```

The `B>` routes with `seg6` are the imported VPN routes. They tell the kernel:
"to reach this subnet, encapsulate with SRv6 and send toward the remote PE."
`C>*` routes are directly connected subnets — no encapsulation needed because
the destination is on a local interface.

## 4.9 Dive Deeper: BGP and L3VPN

- **RFC 4271** — "A Border Gateway Protocol 4 (BGP-4)" — the core BGP
specification
- **RFC 4456** — "BGP Route Reflection" — how route reflectors work
- **RFC 4364** — "BGP/MPLS IP Virtual Private Networks (VPNs)" — the original
L3VPN RFC (MPLS-based, but the BGP concepts are identical for SRv6)
- **RFC 9252** — "BGP Overlay Services Based on Segment Routing over IPv6" —
how BGP carries SRv6 SIDs for VPN services
- **RFC 4360** — "BGP Extended Communities Attribute" — defines Route Targets
- **"Internet Routing Architectures" by Sam Halabi** — excellent BGP primer
- **"MPLS-Enabled Applications" by Ina Minei & Julian Lucek** — covers L3VPN
concepts in depth (applicable to SRv6 despite the MPLS title)
- **FRR BGP documentation** — [https://docs.frrouting.org/en/latest/bgp.html](https://docs.frrouting.org/en/latest/bgp.html)


---

[← Part 3: SRv6](03-srv6.md) | [Back to Index](../LEARNING.md) | [Part 5: VXLAN & EVPN →](05-vxlan-evpn.md)
