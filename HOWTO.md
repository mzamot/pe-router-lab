# How-To: Build an SRv6 + EVPN VXLAN Fabric from Scratch

A step-by-step configuration guide. You start with four blank nodes — one FRR
container (spine) and three AlmaLinux 10 VMs (PE1, PE2, PE3) — connected in a
star topology. By the end, you have a fully operational SRv6 fabric with L3VPN
and L2 VXLAN overlay.

Every command is typed manually. No automation scripts. You will configure each
protocol one at a time, verify it works, then move to the next layer.

**Prerequisites:** You should have read LEARNING.md first. This guide assumes
you understand what each protocol does and why — here, we focus on the exact
commands, what every line and flag does, and how the pieces connect.

---

## Table of Contents

- [Prerequisites — Install Containerlab and Build the VM Image](#prerequisites--install-containerlab-and-build-the-vm-image)
- [Phase 0 — Topology and Addressing Plan](#phase-0--topology-and-addressing-plan)
- [Phase 1 — Linux Plumbing: Namespaces, Veths, VRFs, Bridge, VXLAN](#phase-1--linux-plumbing)
- [Phase 2 — IS-IS: Build the Underlay](#phase-2--is-is)
- [Phase 3 — SRv6: Enable Segment Routing](#phase-3--srv6)
- [Phase 4 — BGP L3VPN: VRF Route Exchange](#phase-4--bgp-l3vpn)
- [Phase 5 — BGP EVPN + VXLAN: The L2 Overlay](#phase-5--bgp-evpn--vxlan)
- [Phase 6 — End-to-End Verification](#phase-6--end-to-end-verification)

---

## Prerequisites — Install Containerlab and Build the VM Image

Before you can deploy the lab, you need two things on your host machine:
**containerlab** (the lab orchestrator) and a **vrnetlab AlmaLinux VM image**
(the virtual machine that each PE node runs inside).

### What is Containerlab?

Containerlab is a tool for building networking labs using containers and VMs.
You define a topology in a YAML file — nodes, links, images — and containerlab
creates everything: it starts containers/VMs, wires them together with virtual
links, and assigns management IPs. When you're done, `containerlab destroy`
tears it all down cleanly.

In this lab, containerlab manages four nodes:
- **Spine:** An FRR container (`quay.io/frrouting/frr:10.5.2`) — this starts
  instantly like any container
- **PE1, PE2, PE3:** AlmaLinux 10 VMs running inside containers via vrnetlab —
  these boot like real VMs (takes ~60-90 seconds)

Containerlab creates the virtual links between the spine's eth1/eth2/eth3 and each
PE's eth1, replicating a physical star topology entirely in software.

### What is vrnetlab?

vrnetlab (VR Network Lab) is a tool that wraps virtual machine disk images
inside container images. This lets containerlab manage VMs the same way it
manages containers — `docker pull`, `docker run`, etc. Under the hood, vrnetlab
launches QEMU inside the container to run the actual VM, and it bridges the
VM's network interfaces to the container's interfaces.

For this lab, we use an AlmaLinux 10 VM image. Each PE starts as a fresh
AlmaLinux VM with SSH access, into which we then deploy the full networking stack
(namespaces, FRR, VXLAN, etc.).

### Host Requirements

- **Linux host** (bare metal or VM with nested virtualization enabled)
- **Docker or Podman** — containerlab uses Docker by default; Podman works with
  `--runtime podman`
- **Root/sudo access** — containerlab creates network namespaces and virtual
  interfaces, which require elevated privileges
- **Resources:** At least 8 GB RAM free (each AlmaLinux VM uses ~1-2 GB) and
  ~15 GB disk for the VM images
- **KVM support** — vrnetlab uses QEMU/KVM for the VMs. Check with:

```bash
lsmod | grep kvm
```

You should see `kvm` and `kvm_intel` (or `kvm_amd`). If not, enable
virtualization in your BIOS/UEFI settings.

### Step 1: Install Containerlab

Containerlab provides an install script that detects your OS and installs the
latest release:

```bash
sudo bash -c "$(curl -sL https://get.containerlab.dev)"
```

**What this does:** Downloads the containerlab binary, installs it to
`/usr/bin/containerlab`, and creates a symlink `clab` for convenience.

Verify the installation:

```bash
containerlab version
```

You should see version 0.55.0 or later. The lab topology uses `generic_vm`
node kind, which requires this minimum version.

**Alternative installation methods:**

```bash
# Using package managers (if you prefer):
# Debian/Ubuntu:
echo "deb [trusted=yes] https://apt.fury.io/netdevops/ /" | \
  sudo tee /etc/apt/sources.list.d/netdevops.list
sudo apt update && sudo apt install containerlab

# Fedora/RHEL/CentOS:
sudo dnf copr enable netdevops/containerlab
sudo dnf install containerlab

# Arch Linux (AUR):
yay -S containerlab-bin
```

### Step 2: Install sshpass

The deployment script uses `sshpass` to non-interactively SSH into the PE VMs
(which have a default password). Install it:

```bash
# Debian/Ubuntu:
sudo apt install sshpass

# Fedora/RHEL:
sudo dnf install sshpass

# Arch Linux:
sudo pacman -S sshpass
```

### Step 3: The vrnetlab AlmaLinux Image

This lab uses a vrnetlab AlmaLinux 10 VM image
(`localhost/vrnetlab/rhel_almalinux:10`). The image should already be built
and available locally. Verify it exists:

```bash
podman images | grep rhel_almalinux
```

You should see:

```
localhost/vrnetlab/rhel_almalinux   10   <hash>   <date>   <size>
```

This `localhost/vrnetlab/rhel_almalinux:10` is exactly what the
`topology.clab.yml` file references for each PE node:

```yaml
pe1:
  kind: generic_vm
  image: localhost/vrnetlab/rhel_almalinux:10
```

### Step 4: Deploy the Lab

With containerlab installed and the VM image built, you can deploy the full
lab. There are two ways:

**Option A: Automated deployment (recommended)**

```bash
cd /path/to/appliance/lab
sudo bash deploy.sh
```

**What `deploy.sh` does:**
1. Runs `containerlab deploy -t topology.clab.yml` — starts the spine (instant) and
   the three PE VMs (each takes ~60-90 seconds to boot)
2. Waits for SSH to become available on each PE VM
3. Copies `scripts/setup-pe.sh` into each VM via SCP
4. Runs the setup script on each PE, which creates the namespaces, veth pairs,
   VRFs, VXLAN bridge, and starts FRR containers — the full networking stack

After ~3-5 minutes, the script outputs SSH connection info and verification
commands.

**Option B: Manual deployment**

If you want to understand each step (or if you're following the HOWTO guide to
configure everything by hand):

```bash
cd /path/to/appliance/lab
sudo containerlab deploy -t topology.clab.yml
```

Wait ~90 seconds for the VMs to boot, then SSH into each PE:

```bash
ssh clab@clab-srv6-lab-pe1    # password: clab@123
ssh clab@clab-srv6-lab-pe2
ssh clab@clab-srv6-lab-pe3
```

From here, you follow the HOWTO guide starting at Phase 1 to configure
everything manually, one command at a time.

### Step 5: Verify the Lab Is Running

```bash
sudo containerlab inspect -t topology.clab.yml
```

This shows the status of all nodes, their management IPs, and their container
IDs. You should see four nodes: `spine`, `pe1`, `pe2`, `pe3`, all in `running`
state.

To check that the spine FRR container is healthy:

```bash
sudo docker exec clab-srv6-lab-spine vtysh -c "show version"
```

To check that a PE VM is reachable:

```bash
ssh clab@clab-srv6-lab-pe1 hostname
```

If all four nodes are running and SSH works to the PEs, you're ready to proceed
to Phase 0.

### Tearing Down the Lab

When you're done, clean everything up:

```bash
sudo containerlab destroy -t topology.clab.yml --cleanup
```

**What this does:** Stops all containers/VMs, removes the virtual links, and
deletes the lab directory (`clab-srv6-lab/`). The `--cleanup` flag also removes
any container images that were pulled. Your vrnetlab AlmaLinux image remains
intact — you don't need to rebuild it next time.

---

## Phase 0 — Topology and Addressing Plan

### Physical Topology

```
                  ┌──────────────┐
                  │    spine     │   FRR 10.5 container
                  │  (kind: linux)│   IS-IS + BGP RR + SRv6
                  └─┬─────┬────┬─┘
                   ╱      │     ╲
             eth1 ╱  eth2 │      ╲ eth3
                 ╱        │       ╲
          ┌──────┐    ┌──────┐    ┌──────┐
          │ PE1  │    │ PE2  │    │ PE3  │   AlmaLinux 10 VMs
          │ eth1 │    │ eth1 │    │ eth1 │   (kind: generic_vm)
          └──────┘    └──────┘    └──────┘
```

**Spine** is an FRR container managed by containerlab. It has three
Ethernet interfaces (eth1, eth2, eth3), each connected to one PE's `eth1`
interface. The spine serves two roles: (1) IS-IS router in the underlay
(providing connectivity between PEs), and (2) BGP route reflector for L3VPN
(reflecting VRF routes between PEs). The spine does NOT participate in EVPN
or VXLAN.

**PE1, PE2, PE3** are AlmaLinux 10 VMs running as vrnetlab nodes in
containerlab. Each has one data interface (`eth1`) connected to the spine. Inside each
PE, we will build two network namespaces (`pe-router` and `l2-vxlan`) and four
veth pairs that connect the namespaces.

### Addressing Plan

Print this out and keep it next to you. Every address in this guide comes from
this table. When a command says "substitute for your node," refer back here.

#### Underlay (IS-IS + SRv6)

| Node | IS-IS NET | Loopback IPv6 | SRv6 Locator | SRv6 Source |
|------|-----------|---------------|--------------|-------------|
| Spine | 49.0000.0000.0000.0001.00 | fc00:0:0:1::1/128 | fd00:30:1::/48 | fc00:0:0:1::1 |
| PE1 | 49.0000.1000.f006.0016.00 | fc00:0:0:1::16/128 | fd00:30:16::/48 | fd00:30:16:: |
| PE2 | 49.0000.1000.f006.0017.00 | fc00:0:0:1::17/128 | fd00:30:17::/48 | fd00:30:17:: |
| PE3 | 49.0000.1000.f006.0018.00 | fc00:0:0:1::18/128 | fd00:30:18::/48 | fd00:30:18:: |

**IS-IS NET** is the router's OSI network address — it encodes the area
(49.0000) and a unique system ID (the middle 6 bytes). All nodes share area
49.0000 so they can form Level-1 adjacencies.

**Loopback IPv6** is the stable address used for BGP peering. It's a /128
(single address) on the loopback interface, always reachable as long as the
router is up.

**SRv6 Locator** is the /48 prefix block assigned to each node. IS-IS
advertises it so the entire fabric knows how to route SRv6 packets toward
the owning node.

**SRv6 Source** is the address placed in the outer IPv6 source field when this
node encapsulates packets with SRv6.

#### BGP L3VPN (AS 65500)

| Node | BGP Router-ID | Peer | Role |
|------|---------------|------|------|
| Spine | 1.1.1.1 | PE1, PE2, PE3 | Route Reflector |
| PE1 | 172.31.250.106 | Spine | Client |
| PE2 | 172.31.250.107 | Spine | Client |
| PE3 | 172.31.250.108 | Spine | Client |

All nodes are in AS 65500 (iBGP). The spine reflects VPN routes between PEs so they
don't need to form a full mesh. The router-ID is a 32-bit unique identifier —
it doesn't need to be a reachable IP address, but using the management IP makes
troubleshooting easier.

#### VRF CMN (Route Target 65004:4)

| Node | RD | lo-cmn | ve-pe-cmn-ovn | ve-pe-cmn-host | ve-pe-cmn-l2 | Static route |
|------|----|--------|---------------|----------------|--------------|--------------|
| PE1 | 172.31.250.106:4 | 10.55.98.6/32 | 10.55.96.21/30 | 10.55.97.21/30 | 10.55.96.149/30 | 10.55.99.0/26 → 10.55.96.150 |
| PE2 | 172.31.250.107:4 | 10.55.98.7/32 | 10.55.96.26/30 | 10.55.97.26/30 | 10.55.96.153/30 | 10.55.99.0/26 → 10.55.96.154 |
| PE3 | 172.31.250.108:4 | 10.55.98.8/32 | 10.55.96.30/30 | 10.55.97.30/30 | 10.55.96.157/30 | 10.55.99.0/26 → 10.55.96.158 |

**RD (Route Distinguisher)** makes each PE's VPN routes unique in BGP. Format:
`router-id:vrf-table-number`.

**Static route** tells the CMN VRF that the machine network (10.55.99.0/26) is
reachable via the VTEP address in the l2-vxlan namespace. This route gets
redistributed into BGP L3VPN and advertised to other PEs.

#### Default Namespace (host side of veths)

| Node | ve-ovn-pe-cmn | ve-host-pe-cmn | ve-home-l2-br (machine net) |
|------|---------------|----------------|------|
| PE1 | 10.55.96.22/30 | 10.55.97.22/30 | 10.55.99.7/26 |
| PE2 | 10.55.96.25/30 | 10.55.97.25/30 | 10.55.99.8/26 |
| PE3 | 10.55.96.29/30 | 10.55.97.29/30 | 10.55.99.9/26 |

#### L2 VXLAN Overlay (EVPN, AS 65501)

| Node | VTEP IP (ve-l2-pe-cmn) | BGP Router-ID | EVPN Role |
|------|------------------------|---------------|-----------|
| PE1 | 10.55.96.150/30 | 1.1.1.6 | Route Reflector |
| PE2 | 10.55.96.154/30 | 1.1.1.7 | Client |
| PE3 | 10.55.96.158/30 | 1.1.1.8 | Client |

- VNI: 330
- VLAN ID: 30
- Anycast gateway: 10.55.99.1/26, MAC aa:dd:cc:00:ff:ee (identical on all PEs)

The EVPN BGP runs in AS 65501 (separate from L3VPN's 65500) because it's a
completely separate BGP instance in a separate namespace (l2-vxlan) with
separate FRR processes.

### Prerequisites

On each AlmaLinux VM:

```bash
sudo dnf install -y ethtool iproute podman
sudo podman pull quay.io/frrouting/frr:10.5.2

for mod in dummy vrf veth vxlan bridge br_netfilter 8021q; do
  sudo modprobe $mod
done
```

**What each package provides:**
- `ethtool` — Used to disable hardware checksum offload on virtual interfaces
  (required because virtual devices use software checksums)
- `iproute` — The `ip` command suite for managing interfaces, routes,
  namespaces, bridges, and VXLAN devices
- `podman` — Container runtime for running FRR containers (used instead of
  Docker for rootless container support)

**What each kernel module does:**
- `dummy` — Creates dummy interfaces (used for SRv6 source address device `sr0`
  and VRF loopbacks like `lo-cmn`)
- `vrf` — Enables VRF (Virtual Routing and Forwarding) support in the kernel,
  allowing separate routing tables per VRF
- `veth` — Enables virtual Ethernet pairs for connecting namespaces
- `vxlan` — Enables VXLAN tunnel device support (encap/decap of VXLAN frames)
- `bridge` — Enables Linux bridge (software switch) functionality
- `br_netfilter` — Allows iptables/nftables to see bridged traffic (needed for
  some firewall and filtering use cases)
- `8021q` — Enables 802.1Q VLAN tagging support (used for the VLAN sub-
  interface on the bridge)

---

## Phase 1 — Linux Plumbing

**Goal:** Build the namespace, veth, VRF, bridge, and VXLAN infrastructure on
each PE. No routing protocols yet — just the wiring.

**What this phase accomplishes:** When finished, each PE will have three
namespaces (default, pe-router, l2-vxlan) connected by veth pairs, with VRFs
configured in pe-router and a VXLAN bridge in l2-vxlan. Local connectivity
across veths will work, but remote PE connectivity won't (that requires IS-IS,
BGP, and EVPN in later phases).

> Do this on ALL THREE PEs. Substitute the correct addresses from the table
> above for each node. The commands below show PE1's values — the comments
> note what changes per node.

### 1.1 Create the pe-router Namespace

```bash
sudo ip netns add pe-router
```

**What this does:** Creates a new network namespace called `pe-router`. The
`ip netns add` command creates a bind mount at `/run/netns/pe-router` that
persists the namespace even when no process is running inside it. The new
namespace starts with ONLY a loopback interface (`lo`), and it's DOWN.

```bash
sudo ip netns exec pe-router ip link set lo up
```

**What this does:** `ip netns exec pe-router` runs the following command inside
the pe-router namespace. `ip link set lo up` brings the loopback interface to
the UP state. The loopback MUST be up for IS-IS and BGP to function — IS-IS
uses it as its router identity, and BGP uses it as the session source address.
Without this, FRR daemons running in this namespace would have no loopback to
bind to.

```bash
sudo ip netns exec pe-router ip addr add fc00:0:0:1::16/128 dev lo
#                                        ^^^^^^^^^^^^^^^^
#                         PE2: fc00:0:0:1::17/128  PE3: fc00:0:0:1::18/128
```

**What this does:** Assigns the IPv6 loopback address to the `lo` interface.
The `/128` prefix means this is a single host address (the IPv6 equivalent of
a `/32` in IPv4). This address serves as:
- The IS-IS router identity (advertised in the IS-IS database so all other
  routers know how to reach this node)
- The BGP update-source (BGP sessions are established TO and FROM this address)
- A stable, always-up endpoint that doesn't depend on any physical interface

**Why `fc00::`?** The `fc00::/7` block is IPv6 Unique Local Address (ULA)
space — the equivalent of RFC1918 private addresses in IPv4. It's used here
because these addresses are internal to the fabric and never need to be
globally routable.

### 1.2 Enable Forwarding and SRv6

```bash
sudo ip netns exec pe-router sysctl -w \
  net.ipv6.conf.all.seg6_enabled=1 \
  net.ipv6.conf.all.forwarding=1 \
  net.ipv4.conf.all.forwarding=1 \
  net.ipv4.conf.all.rp_filter=0 \
  net.ipv4.conf.default.rp_filter=0 \
  net.ipv6.seg6_flowlabel=1
```

**What `sysctl -w` does:** Writes a value to a kernel tunable parameter at
runtime. The `-w` flag means "write." Each sysctl controls a specific kernel
behavior. The `ip netns exec pe-router` prefix ensures these are set ONLY in
the pe-router namespace — the default namespace keeps its own values.

**Line-by-line explanation:**

| Sysctl | Value | What It Does | Why We Need It |
|--------|-------|-------------|----------------|
| `net.ipv6.conf.all.seg6_enabled` | 1 | Enables SRv6 (Segment Routing over IPv6) processing globally in this namespace. When enabled, the kernel checks incoming IPv6 packets against the local SID table and performs SRv6 actions (End, End.DT4, etc.) on matching packets. | Without this, the kernel would treat SRv6 SIDs as normal IPv6 addresses and never perform decapsulation or VRF delivery. SRv6 would be completely non-functional. |
| `net.ipv6.conf.all.forwarding` | 1 | Enables IPv6 packet forwarding. When a packet arrives that isn't destined for a local address, the kernel looks up the destination in the routing table and forwards it to the next hop instead of dropping it. | This namespace IS a router. Without forwarding, it would only accept packets for its own addresses and silently drop everything else. IS-IS, BGP, SRv6 transit — all require forwarding. |
| `net.ipv4.conf.all.forwarding` | 1 | Same as above but for IPv4. Enables the kernel to forward IPv4 packets between interfaces. | VRF traffic (CMN, EDN) is IPv4. Without this, IPv4 packets entering one veth wouldn't be forwarded out another veth or encapsulated with SRv6. |
| `net.ipv4.conf.all.rp_filter` | 0 | Disables Reverse Path Filtering (RPF) globally. RPF is a security feature that drops packets if they arrive on an interface that the kernel wouldn't use to send to the packet's source address. | VRF + SRv6 creates asymmetric routing: a packet might arrive on eth1 (after SRv6 decap) but the return path uses a veth. RPF would drop these packets because the arrival interface doesn't match the expected return path. Disabling RPF prevents these false drops. |
| `net.ipv4.conf.default.rp_filter` | 0 | Sets the default RPF value for newly created interfaces. Without this, new interfaces (veths we create later) would inherit rp_filter=1 from the system default. | Ensures veths and other interfaces created after this point also have RPF disabled, without needing to set it per-interface after creation. |
| `net.ipv6.seg6_flowlabel` | 1 | When creating SRv6 outer IPv6 headers, copy the flow information from the inner packet into the outer IPv6 Flow Label field. | The Flow Label is used by routers for ECMP (Equal-Cost Multi-Path) hashing. Without this, all SRv6 packets from the same source would have the same flow label and would all take the same path, defeating ECMP load balancing. With it enabled, different inner flows get different outer flow labels and are distributed across paths. |

### 1.3 Create the SRv6 Dummy and VRFs

```bash
sudo ip netns exec pe-router ip link add sr0 type dummy
sudo ip netns exec pe-router ip link set sr0 up
```

**What this does:** Creates a `dummy` interface called `sr0`. A dummy interface
is a virtual interface that is always UP and has no physical backing — think of
it as a programmable loopback. It's used here as the SRv6 source address
device. FRR configures the SRv6 source address on this device, and the kernel
uses it as the source for all SRv6-encapsulated packets. We use a dedicated
dummy rather than `lo` to keep the SRv6 source address separate from the IS-IS/
BGP loopback address (they're in different IPv6 prefixes: `fd00:30:X::` vs
`fc00:0:0:1::X`).

```bash
sudo ip netns exec pe-router ip link add CMN type vrf table 4
sudo ip netns exec pe-router ip link set CMN up
```

**What this does:** Creates a VRF device called `CMN` backed by kernel routing
table 4. The `ip link add CMN type vrf table 4` command does two things: (1)
creates a virtual interface named `CMN` that acts as the VRF master device, and
(2) associates routing table 4 with this VRF. Any interface enslaved to `CMN`
(via `ip link set <iface> master CMN`) will use table 4 for all routing
decisions. `ip link set CMN up` activates the VRF — without this, enslaved
interfaces won't forward traffic.

**Why table 4?** The table number is arbitrary but must be unique per VRF. We
use 4 for CMN and 2 for EDN. These numbers appear in the SRv6 `End.DT4`
behavior (`vrftable 4`) and in the BGP config. The default routing table is
always 254 (also called "main").

```bash
sudo ip netns exec pe-router ip link add EDN type vrf table 2
sudo ip netns exec pe-router ip link set EDN up
```

**What this does:** Same as above but for the EDN (External Data Network) VRF
using table 2. EDN is a placeholder for external-facing services — in this lab
it's mostly empty, but the infrastructure is ready for it.

```bash
sudo ip netns exec pe-router sysctl -w net.vrf.strict_mode=1
```

**What this does:** Enables VRF strict mode, which enforces that each interface
can belong to AT MOST one VRF. Without strict mode, it's possible to
accidentally enslave an interface to multiple VRFs, which causes undefined
routing behavior. Strict mode prevents this by erroring if you try to assign
an interface that's already in a VRF. Think of it as a safety guardrail.

### 1.4 Create the CMN Loopback

```bash
sudo ip netns exec pe-router ip link add lo-cmn type dummy
```

**What this does:** Creates another dummy interface, this time for the CMN VRF.
Every VRF benefits from having its own loopback — a stable, always-up address
within the VRF. This is useful for management access to the VRF and as a
source address for VRF-originated traffic.

```bash
sudo ip netns exec pe-router ip link set lo-cmn master CMN
```

**What this does:** Enslaves `lo-cmn` to the CMN VRF. After this, `lo-cmn`'s
routing decisions use table 4 (CMN). The `master CMN` syntax means "this
interface is owned by VRF CMN."

```bash
sudo ip netns exec pe-router ip link set lo-cmn up
sudo ip netns exec pe-router ip addr add 10.55.98.6/32 dev lo-cmn
#                                        ^^^^^^^^^^^
#                              PE2: 10.55.98.7/32  PE3: 10.55.98.8/32
```

**What this does:** Brings `lo-cmn` up and assigns it a `/32` address (single
host). This address is advertised via BGP L3VPN and can be used to reach the
CMN VRF for management purposes. The `/32` is standard for loopbacks because
there are no other hosts on this "link."

### 1.5 Move the Fabric Interface

```bash
sudo ip link set eth1 netns pe-router
```

**What this does:** Moves the physical NIC `eth1` from the default namespace
into the pe-router namespace. After this command:
- `eth1` disappears from `ip link show` in the default namespace
- `eth1` appears in `sudo ip netns exec pe-router ip link show`
- The default namespace loses ALL direct physical network access
- ALL traffic in/out of the machine MUST flow through pe-router

This is a fundamental design decision: pe-router controls all external
communication. The host (default namespace) can only reach the outside world
through the veth pairs that connect it to pe-router's VRFs.

```bash
sudo ip netns exec pe-router ip link set eth1 mtu 9000
```

**What this does:** Sets the MTU (Maximum Transmission Unit) on `eth1` to
9000 bytes (jumbo frames). Standard Ethernet MTU is 1500 bytes. With double
encapsulation (SRv6 adds ~40 bytes, VXLAN adds ~50 bytes = 90 bytes total),
a 1500-byte inner packet becomes ~1590 bytes. If the fabric link had MTU 1500,
these packets would either be fragmented (bad for performance — the receiver
must reassemble, consuming CPU) or dropped (if Don't Fragment is set). MTU 9000
provides ample headroom: even with 90 bytes of overhead, you can carry inner
frames up to ~8910 bytes without fragmentation.

```bash
sudo ip netns exec pe-router ip link set eth1 up
```

**What this does:** Brings `eth1` up in the pe-router namespace. The interface
was likely down after the namespace move. IS-IS will run on this interface to
form an adjacency with the spine.

```bash
sudo ip netns exec pe-router sysctl -w \
  net.ipv4.conf.eth1.rp_filter=0 \
  net.ipv4.conf.CMN.rp_filter=0 \
  net.ipv4.conf.EDN.rp_filter=0 \
  net.ipv4.conf.lo.rp_filter=0 \
  net.ipv4.conf.lo-cmn.rp_filter=0 \
  net.ipv4.conf.sr0.rp_filter=0 \
  net.ipv6.conf.eth1.seg6_enabled=1
```

**What this does:** Disables RPF on every interface that exists so far in the
pe-router namespace, and enables SRv6 processing specifically on `eth1`.

**Why per-interface RPF disable?** Linux RPF is evaluated per-interface using
the MAXIMUM of the interface-specific value and the `all` value. So even though
we set `net.ipv4.conf.all.rp_filter=0` earlier, if an interface's own setting
is non-zero, RPF would still be active on that interface. By explicitly setting
each interface to 0, we ensure RPF is truly disabled everywhere.

**Why per-interface SRv6 enable on eth1?** While `net.ipv6.conf.all.seg6_enabled=1`
enables global SRv6 processing, some kernel versions also require per-interface
enablement for SRv6 to process packets arriving on that specific interface.
`eth1` is the fabric interface where all SRv6 packets arrive from the spine, so it
must have SRv6 enabled.

### 1.6 SRv6 Policy Routing

```bash
sudo ip netns exec pe-router ip -6 rule add to fd00::/16 lookup 254 pref 999
sudo ip netns exec pe-router ip -6 rule add to fd01::/16 lookup 254 pref 999
```

**What this does:** Adds IPv6 policy routing rules. Policy rules are checked
BEFORE the routing table and control WHICH routing table the kernel uses for a
given packet. These rules say:

- `to fd00::/16` — for any IPv6 packet destined to an address in `fd00::/16`
  (which covers all SRv6 locators like `fd00:30:16::/48`)
- `lookup 254` — use routing table 254 (the main/default table, where IS-IS
  routes live)
- `pref 999` — priority 999 (lower = checked first; this ensures it's checked
  before VRF rules which typically have priority 1000)

The `fd01::/16` rule is a safety net for any future SRv6 addresses in that
range.

**Why this is necessary:** When VRF strict mode is enabled, packets originating
from a VRF context (e.g., the CMN VRF trying to SRv6-encapsulate a packet)
initially look up routes in the VRF table (table 4). But SRv6 locator routes
(`fd00:30:17::/48` via eth1) are in the main table (254), not in the VRF table.
Without these policy rules, the kernel would look in table 4, not find the SRv6
locator route, and drop the packet. These rules force the kernel to always
check table 254 for SRv6 destinations, regardless of VRF context.

### 1.7 Create Veth Pairs

Each veth connects two namespaces. We create the pair in the default namespace,
then move one end to the target namespace.

**Node network veth (default ↔ pe-router CMN):**

This pair provides the host with a direct routed path to the CMN VRF's
infrastructure address space. The "ovn" in the name is a legacy artifact. Today,
this veth provides the host with
direct L3 access to CMN infrastructure addresses (10.55.96.0/24) — VTEP IPs,
pe-router CMN interfaces — via a static route, without hairpinning through the
VXLAN bridge.

```bash
sudo ip link add ve-ovn-pe-cmn mtu 8900 type veth peer name ve-pe-cmn-ovn
```

**What this does:** Creates a veth pair with two ends: `ve-ovn-pe-cmn` and
`ve-pe-cmn-ovn`. Both start in the default namespace. The `mtu 8900` sets the
MTU at creation time. MTU 8900 (not 9000) because this veth carries traffic
that may later be encapsulated with VXLAN (~50 bytes) and SRv6 (~40 bytes).
8900 + 90 = 8990, which fits within the 9000-byte MTU on eth1 with room to
spare. Setting MTU at creation avoids a window where the interface is up with
the wrong MTU.

```bash
sudo ip link set ve-pe-cmn-ovn netns pe-router
```

**What this does:** Moves the `ve-pe-cmn-ovn` end of the veth into the
pe-router namespace. After this, `ve-ovn-pe-cmn` is in the default namespace
and `ve-pe-cmn-ovn` is in pe-router. They're still connected — traffic sent
into one emerges from the other, crossing the namespace boundary.

```bash
sudo ip netns exec pe-router ip link set ve-pe-cmn-ovn master CMN
```

**What this does:** Enslaves `ve-pe-cmn-ovn` to the CMN VRF. All routing
decisions for traffic on this interface now use table 4 (CMN). This is how the
node's host network traffic enters the CMN VRF.

```bash
sudo ip netns exec pe-router ip link set ve-pe-cmn-ovn mtu 8900
sudo ip netns exec pe-router ip link set ve-pe-cmn-ovn up
sudo ip netns exec pe-router ip addr add 10.55.96.21/30 dev ve-pe-cmn-ovn
#                                        ^^^^^^^^^^^^^
#                              PE2: 10.55.96.26/30  PE3: 10.55.96.30/30
```

**What this does:** Sets MTU (may have been reset during namespace move on some
kernel versions), brings the interface up, and assigns an IP address. The `/30`
subnet provides exactly 2 usable addresses: `.21` on the pe-router side and
`.22` on the default namespace side. This creates a point-to-point link between
the two namespaces.

```bash
sudo ip link set ve-ovn-pe-cmn up
sudo ip addr add 10.55.96.22/30 dev ve-ovn-pe-cmn
#                ^^^^^^^^^^^^^
#      PE2: 10.55.96.25/30  PE3: 10.55.96.29/30
```

**What this does:** Configures the default namespace side. Now both ends have
IP addresses on the same `/30` subnet, and they can ping each other. This
requires no routing protocol — it's a directly connected link.

**Host veth (default ↔ pe-router CMN):**

This pair carries dedicated host management traffic. It's structurally
identical to the CMN infrastructure veth but uses different addresses.

```bash
sudo ip link add ve-host-pe-cmn mtu 8900 type veth peer name ve-pe-cmn-host

sudo ip link set ve-pe-cmn-host netns pe-router
sudo ip netns exec pe-router ip link set ve-pe-cmn-host master CMN
sudo ip netns exec pe-router ip link set ve-pe-cmn-host mtu 8900
sudo ip netns exec pe-router ip link set ve-pe-cmn-host up
sudo ip netns exec pe-router ip addr add 10.55.97.21/30 dev ve-pe-cmn-host
#                                        ^^^^^^^^^^^^^
#                              PE2: 10.55.97.26/30  PE3: 10.55.97.30/30

sudo ip link set ve-host-pe-cmn up
sudo ip addr add 10.55.97.22/30 dev ve-host-pe-cmn
#                ^^^^^^^^^^^^^
#      PE2: 10.55.97.25/30  PE3: 10.55.97.29/30
```

**Why two veths to the same VRF?** The "ovn" veth and host management veth
both connect to the CMN VRF but serve different roles. Separating them allows
different routing or policy per traffic type. Each path can have independent
QoS or firewall rules.

```bash
sudo ip netns exec pe-router sysctl -w \
  net.ipv4.conf.ve-pe-cmn-ovn.rp_filter=0 \
  net.ipv4.conf.ve-pe-cmn-host.rp_filter=0
```

**What this does:** Disables RPF on the newly created veth interfaces inside
pe-router. These interfaces didn't exist when we set the `all` and `default`
RPF values, so we must set them explicitly.

### 1.8 Create the l2-vxlan Namespace

```bash
sudo ip netns add l2-vxlan
sudo ip netns exec l2-vxlan ip link set lo up
```

**What this does:** Creates the second namespace (`l2-vxlan`) and brings up its
loopback. This namespace will host the VXLAN bridge, the VXLAN tunnel device,
and the EVPN FRR instance. It's separate from pe-router because it needs its
own routing table (a simple default route pointing to pe-router's CMN VRF) and
its own FRR process (running BGP EVPN with a different AS number).

### 1.9 Create the L2-PE Veth (l2-vxlan ↔ pe-router CMN)

This is the most critical veth — it carries ALL VTEP traffic between the VXLAN
namespace and the PE router's CMN VRF. EVPN BGP sessions, VXLAN-encapsulated
frames, and ARP between VTEPs all flow through this single veth pair.

```bash
sudo ip link add ve-l2-pe-cmn mtu 9000 type veth peer name ve-pe-cmn-l2
```

**What this does:** Creates the veth pair. Note MTU 9000 (not 8900 like the
other veths). This veth carries VXLAN-encapsulated traffic that will be SRv6-
encapsulated on the OTHER side (in pe-router). The VXLAN packet is ~50 bytes
overhead, and SRv6 adds ~40 bytes AFTER this veth — so the traffic on this
veth only has VXLAN overhead, not SRv6. 9000 - 50 = 8950 bytes of inner frame
capacity, which is plenty.

```bash
sudo ip link set ve-l2-pe-cmn netns l2-vxlan
sudo ip link set ve-pe-cmn-l2 netns pe-router
```

**What this does:** Moves each end to its target namespace. Unlike the previous
veths (where one end stayed in default), BOTH ends of this pair are moved:
`ve-l2-pe-cmn` goes to l2-vxlan, `ve-pe-cmn-l2` goes to pe-router. The
default namespace is just the creation point.

```bash
sudo ip netns exec pe-router ip link set ve-pe-cmn-l2 master CMN
sudo ip netns exec pe-router ip link set ve-pe-cmn-l2 mtu 9000
sudo ip netns exec pe-router ip link set ve-pe-cmn-l2 up
sudo ip netns exec pe-router ip addr add 10.55.96.149/30 dev ve-pe-cmn-l2
#                                        ^^^^^^^^^^^^^^
#                              PE2: 10.55.96.153/30  PE3: 10.55.96.157/30
```

**What this does:** Configures the pe-router end. `master CMN` enslaves it to
the CMN VRF. The IP address (10.55.96.149) becomes the gateway for the
l2-vxlan namespace's default route.

```bash
sudo ip netns exec l2-vxlan ip link set ve-l2-pe-cmn mtu 9000
sudo ip netns exec l2-vxlan ip link set ve-l2-pe-cmn up
sudo ip netns exec l2-vxlan ip addr add 10.55.96.150/30 dev ve-l2-pe-cmn
#                                       ^^^^^^^^^^^^^^
#                             PE2: 10.55.96.154/30  PE3: 10.55.96.158/30
```

**What this does:** Configures the l2-vxlan end. The IP address (10.55.96.150)
IS the VTEP address — this is the source IP for all VXLAN-encapsulated packets
from this PE. When the VXLAN device sends a packet, it uses this address as
the outer source IP.

```bash
sudo ip netns exec pe-router sysctl -w net.ipv4.conf.ve-pe-cmn-l2.rp_filter=0
```

**What this does:** Disables RPF on the new interface. Traffic arriving here
comes from the l2-vxlan namespace with source addresses that may not have a
reverse route through this interface (asymmetric routing via SRv6).

```bash
sudo ip netns exec l2-vxlan ip route add default via 10.55.96.149
#                                                    ^^^^^^^^^^^
#                                          PE2: 10.55.96.153  PE3: 10.55.96.157
```

**What this does:** Adds a default route in the l2-vxlan namespace pointing to
the pe-router's CMN VRF interface. This is how the VTEP reaches the outside
world. When l2-vxlan needs to send a VXLAN packet to a remote VTEP (e.g.,
10.55.96.154 on PE2), it sends it to 10.55.96.149 (pe-router's CMN VRF), which
then:
1. Looks up 10.55.96.154 in the CMN routing table
2. Finds a BGP VPN route with an SRv6 SID
3. SRv6-encapsulates and sends via the underlay

This single default route is the link between the VXLAN world and the SRv6
fabric.

### 1.10 Build the VXLAN Bridge

```bash
sudo ip netns exec l2-vxlan ip link add br0 type bridge \
  vlan_filtering 1 vlan_default_pvid 0
```

**What this does:** Creates a Linux bridge called `br0` in the l2-vxlan
namespace. A bridge is a software Layer 2 switch — it learns MAC addresses
from incoming frames and forwards frames to the correct port based on its FDB
(Forwarding Database).

- `type bridge` — specifies this is a bridge device
- `vlan_filtering 1` — enables 802.1Q VLAN-aware mode. Without this, the bridge
  treats all frames as untagged and doesn't understand VLANs. With it, you can
  assign VLANs to each port and the bridge only forwards frames to ports that
  share the same VLAN. This is essential for mapping VLANs to VNIs.
- `vlan_default_pvid 0` — sets the default Port VLAN ID (PVID) to 0, which
  effectively means "no default VLAN." Without this, the bridge auto-assigns
  PVID 1 to every port, and untagged frames get VLAN 1. We set it to 0 so we
  can explicitly assign VLAN 30 to each port, avoiding any accidental VLAN 1
  traffic.

```bash
sudo ip netns exec l2-vxlan ip link set br0 mtu 8854
sudo ip netns exec l2-vxlan ip link set br0 up
```

**What this does:** Sets the bridge MTU to 8854 and brings it up. The MTU is
smaller than the veth MTU (9000) because the bridge's ports include vxlan0,
which adds VXLAN headers when sending. The effective MTU for the bridge
should account for VXLAN overhead to avoid fragmentation within the bridge.

```bash
sudo ip netns exec l2-vxlan bridge vlan add vid 30 dev br0 self
```

**What this does:** Adds VLAN 30 to the bridge itself (the `self` keyword means
"the bridge device," not a port). This tells the bridge that VLAN 30 is a
valid VLAN. Frames tagged with VLAN 30 will be processed; frames with unknown
VLANs will be dropped.

### 1.11 Create the VXLAN Device

```bash
sudo ip netns exec l2-vxlan ip link add vxlan0 type vxlan \
  id 330 dstport 4789 local 10.55.96.150 ttl 64 \
  nolearning udpcsum udp6zerocsumrx
#                          ^^^^^^^^^^^
#                PE2: 10.55.96.154  PE3: 10.55.96.158
```

**What this does:** Creates the VXLAN tunnel device. This is the most complex
single command in the setup. Each flag:

| Flag | What It Does | Why It's Needed |
|------|-------------|-----------------|
| `type vxlan` | Creates a VXLAN tunnel device | Specifies this is a VXLAN interface, not a regular Ethernet device |
| `id 330` | Sets the VNI (VXLAN Network Identifier) to 330 | This uniquely identifies the virtual L2 network. All VTEPs using VNI 330 share the same broadcast domain. The EVPN `advertise-all-vni` config discovers this VNI and advertises it. |
| `dstport 4789` | Sets the UDP destination port for VXLAN packets to 4789 | 4789 is the IANA standard port for VXLAN (RFC 7348). Remote VTEPs listen on this port. Using the standard port ensures interoperability with other VXLAN implementations. |
| `local 10.55.96.150` | Sets the source IP for outgoing VXLAN packets | This is the VTEP address. When this device encapsulates a frame, it puts this IP in the outer header's source field. Remote VTEPs use this address to send traffic back. |
| `ttl 64` | Sets the outer IP TTL (Time-to-Live) to 64 | TTL prevents routing loops — each router decrements the TTL and drops the packet when it reaches 0. 64 is the standard starting value. Without this, some kernels use TTL 0, which would cause packets to be dropped immediately. |
| `nolearning` | Disables data-plane MAC learning on the VXLAN device | In traditional VXLAN, the device learns remote MACs from incoming traffic (flood-and-learn). We disable this because EVPN handles MAC learning via the BGP control plane. Data-plane learning would conflict with EVPN and cause race conditions. |
| `udpcsum` | Enables UDP checksum calculation on transmitted VXLAN packets | Some NICs and middleboxes drop UDP packets with zero checksum. Enabling checksums ensures compatibility. The cost is minimal CPU overhead. |
| `udp6zerocsumrx` | Accepts incoming UDP-over-IPv6 packets with zero checksum | When VXLAN traffic crosses the SRv6 fabric, it's encapsulated in IPv6. Some implementations don't compute UDP checksums for IPv6-encapsulated VXLAN. This flag prevents the kernel from dropping those packets. |

```bash
sudo ip netns exec l2-vxlan ip link set vxlan0 addrgenmode none
```

**What this does:** Disables automatic IPv6 address generation on `vxlan0`. By
default, Linux generates a link-local IPv6 address on every interface. For
`vxlan0`, this is unnecessary (VXLAN uses IPv4 for encapsulation in this setup)
and could cause unwanted IPv6 neighbor discovery traffic on the VXLAN tunnel.

```bash
sudo ip netns exec l2-vxlan ip link set vxlan0 master br0
```

**What this does:** Adds `vxlan0` as a port (slave) of the bridge `br0`. Now
the bridge can forward frames to/from the VXLAN tunnel. When a frame needs to
go to a remote PE, the bridge sends it to the `vxlan0` port, which
encapsulates it in VXLAN and transmits it to the remote VTEP.

```bash
sudo ip netns exec l2-vxlan ip link set vxlan0 mtu 8854
sudo ip netns exec l2-vxlan ip link set vxlan0 up
```

**What this does:** Sets MTU to match the bridge and brings the device up.

```bash
sudo ip netns exec l2-vxlan bridge link set dev vxlan0 \
  neigh_suppress on learning off
```

**What this does:** Configures bridge-level settings for the `vxlan0` port.

- `neigh_suppress on` — Enables ARP/ND suppression on this bridge port. When
  the bridge receives an ARP request for a host behind a remote VTEP, instead
  of flooding the ARP to all VTEPs, the bridge answers the ARP locally using
  information learned from EVPN Type-2 routes (which include both MAC and IP).
  This dramatically reduces broadcast traffic on the VXLAN overlay.

- `learning off` — Disables MAC learning on this bridge port (the bridge-level
  complement to `nolearning` on the VXLAN device). The bridge won't add FDB
  entries from data-plane traffic on this port. All FDB entries come from EVPN
  (installed by FRR's zebra daemon). This ensures the FDB is always consistent
  with EVPN's view of the network.

```bash
sudo ip netns exec l2-vxlan bridge vlan add dev vxlan0 vid 30 pvid untagged master
```

**What this does:** Configures VLAN 30 on the `vxlan0` bridge port.

- `vid 30` — This port carries VLAN 30 traffic
- `pvid` — Sets VLAN 30 as the Port VLAN ID (PVID). When an untagged frame
  arrives on this port, it's internally tagged with VLAN 30. Since VXLAN
  frames are inherently untagged (the VLAN is encoded in the VNI, not in
  the Ethernet header), this is how VNI 330 traffic maps to VLAN 30.
- `untagged` — When the bridge sends a frame out this port, it strips the
  VLAN 30 tag. VXLAN encapsulation uses VNI (not VLAN tags) to identify the
  network, so the VLAN tag is removed before encapsulation.
- `master` — Apply this VLAN configuration to the bridge (not the device
  itself). The `master` keyword tells `bridge vlan` that the VLAN is being
  configured on the bridge's port, not on the device directly.

### 1.12 Create the VLAN Sub-interface and Anycast Gateway

```bash
sudo ip netns exec l2-vxlan ip link add vlan30 link br0 type vlan id 30
```

**What this does:** Creates a VLAN sub-interface `vlan30` on top of `br0`. A
VLAN sub-interface receives only frames tagged with the specified VLAN (30 in
this case). This gives us an L3 interface for VLAN 30 — a place where we can
assign an IP address and enable routing for this VLAN.

The `link br0` means this sub-interface is attached to the bridge. It sees all
VLAN 30 traffic that flows through the bridge.

```bash
sudo ip netns exec l2-vxlan ip link set vlan30 mtu 8804
sudo ip netns exec l2-vxlan ip link set vlan30 up
```

**What this does:** Sets MTU (smaller than bridge MTU to account for 802.1Q
VLAN tag overhead of 4 bytes: 8854 - 4 = 8850, rounded for alignment to 8804)
and activates the interface.

```bash
sudo ip netns exec l2-vxlan ip link add vlan-30-ednagw \
  link vlan30 type macvlan mode private
```

**What this does:** Creates a macvlan interface on top of `vlan30`. A macvlan
is a virtual interface that shares the physical (or virtual) interface but has
its own MAC address and IP address. It's like plugging a second NIC into the
same switch port.

- `link vlan30` — The macvlan rides on top of the vlan30 interface
- `type macvlan` — Creates a macvlan device
- `mode private` — The macvlan cannot communicate with the parent interface
  or other macvlans on the same parent. This prevents routing loops between
  the anycast gateway and the bridge.

**Why macvlan instead of assigning the IP directly to vlan30?** Because we need
a specific MAC address (the anycast MAC) that's the same on all PEs. The vlan30
interface has whatever MAC the kernel assigned (unique per PE). A macvlan lets
us set an explicit MAC address while keeping vlan30's original MAC for bridge
operations.

```bash
sudo ip netns exec l2-vxlan ip link set vlan-30-ednagw address aa:dd:cc:00:ff:ee
```

**What this does:** Sets the MAC address of the anycast gateway to
`aa:dd:cc:00:ff:ee`. This EXACT same MAC is configured on PE1, PE2, and PE3.
When any host in the machine network ARPs for the gateway (10.55.99.1), it
receives an ARP reply with this MAC — regardless of which PE it's on. This
means host ARP caches never go stale when traffic is handled by different PEs.

```bash
sudo ip netns exec l2-vxlan ip link set vlan-30-ednagw up
sudo ip netns exec l2-vxlan ip addr add 10.55.99.1/26 dev vlan-30-ednagw
```

**What this does:** Brings up the anycast gateway and assigns its IP address.
`10.55.99.1/26` is the gateway address for the machine network
(`10.55.99.0/26`). The `/26` mask means 64 addresses (0–63), with .1 as the
gateway and .7/.8/.9 as the PE hosts.

```bash
sudo ip netns exec l2-vxlan bridge fdb add aa:dd:cc:00:ff:ee dev br0 self local
```

**What this does:** Adds a static FDB (Forwarding Database) entry in the bridge
for the anycast MAC address.

- `aa:dd:cc:00:ff:ee` — the MAC we're adding
- `dev br0` — on the bridge itself
- `self` — this is a bridge-local entry (not on a specific port)
- `local` — marks this MAC as locally owned. The bridge won't forward frames
  destined to this MAC out any port — it delivers them locally. Without this
  entry, the bridge might try to flood frames for the anycast MAC to all ports
  (including vxlan0, sending them to remote VTEPs unnecessarily).

### 1.13 Create the Home Veth (default ↔ l2-vxlan bridge)

This veth connects the host's machine network interface to the VXLAN bridge.
It's the entry point for host traffic into the L2 overlay.

```bash
sudo ip link add ve-home-l2-br mtu 8800 type veth peer name ve-l2-br-home
```

**What this does:** Creates the veth pair. MTU 8800 is slightly smaller than
the bridge MTU (8854) to provide margin for any bridge-internal overhead.

```bash
sudo ip link set ve-l2-br-home netns l2-vxlan
```

**What this does:** Moves the bridge end into the l2-vxlan namespace.

```bash
sudo ip netns exec l2-vxlan ip link set ve-l2-br-home master br0
```

**What this does:** Adds `ve-l2-br-home` as a port on `br0`. Now the bridge
has two ports: `vxlan0` (for remote traffic) and `ve-l2-br-home` (for local
host traffic). The bridge switches frames between them based on MAC addresses.

```bash
sudo ip netns exec l2-vxlan ip link set ve-l2-br-home mtu 8800
sudo ip netns exec l2-vxlan ip link set ve-l2-br-home up
```

**What this does:** Sets MTU and activates the interface.

```bash
sudo ip netns exec l2-vxlan bridge vlan add dev ve-l2-br-home vid 30 \
  pvid untagged master
```

**What this does:** Assigns VLAN 30 to the `ve-l2-br-home` bridge port, with
the same `pvid untagged` settings as `vxlan0`. This means:
- Incoming untagged frames from the host are internally tagged with VLAN 30
- Outgoing frames to the host have the VLAN 30 tag stripped (sent untagged)
- The host doesn't need to know about VLANs at all — it just sends/receives
  normal Ethernet frames

### 1.14 l2-vxlan Sysctls and Checksum Offload

```bash
sudo ip netns exec l2-vxlan sysctl -w \
  net.ipv4.conf.all.rp_filter=0 \
  net.ipv4.conf.default.rp_filter=0 \
  net.ipv4.conf.all.accept_local=1 \
  net.ipv4.conf.all.proxy_arp=1 \
  net.ipv4.conf.br0.proxy_arp=1 \
  net.ipv4.conf.all.send_redirects=0
```

**Line-by-line explanation:**

| Sysctl | What It Does | Why |
|--------|-------------|-----|
| `rp_filter=0` (all + default) | Disables reverse-path filtering | Same reason as pe-router: asymmetric routing paths from VXLAN traffic |
| `accept_local=1` | Allows accepting packets with a source IP that matches a local address | The anycast gateway IP (10.55.99.1) is the same on all PEs. Without this, a packet from PE2's anycast gateway (10.55.99.1) arriving at PE1 would be dropped because PE1 also owns 10.55.99.1. `accept_local` prevents this. |
| `proxy_arp=1` (all + br0) | Enables proxy ARP — the kernel answers ARP requests on behalf of other hosts | When a host on the bridge ARPs for an IP that the kernel knows about (from EVPN or local routes), the kernel responds with its own MAC. This works with `neigh_suppress` to reduce broadcast traffic. |
| `send_redirects=0` | Disables ICMP Redirects | Normally, if a router receives a packet on the same interface it would forward it out, it sends an ICMP Redirect telling the sender to use a different gateway. In a bridge + VXLAN setup, this can cause incorrect redirect messages because the bridge/VXLAN path isn't a simple router hop. Disabling prevents confusion. |

```bash
for dev in br0 vxlan0 vlan30 vlan-30-ednagw ve-l2-br-home ve-l2-pe-cmn; do
  sudo ip netns exec l2-vxlan ethtool -K $dev rx off tx off 2>/dev/null || true
done
```

**What this does:** Disables hardware checksum offload on every virtual
interface in the l2-vxlan namespace.

- `ethtool -K $dev rx off tx off` — Disables receive (`rx`) and transmit (`tx`)
  checksum offloading
- `2>/dev/null || true` — Suppresses errors (some virtual devices don't support
  ethtool) and ensures the loop continues even if a command fails

**Why this is critical:** Hardware checksum offload tells the kernel "don't
compute checksums in software — the NIC hardware will do it." But virtual
interfaces (veth, vxlan, bridge) have no hardware. When offload is enabled on
a virtual device, the kernel skips checksum computation, and the receiving end
sees an invalid (or zero) checksum and DROPS the packet. This is one of the
most common causes of mysterious packet loss in virtual networking setups.
Disabling offload forces the kernel to compute checksums in software, which
is correct for virtual interfaces.

### 1.15 Configure the Machine Network (Default Namespace)

```bash
sudo ip link set ve-home-l2-br mtu 8800
sudo ip link set ve-home-l2-br up
sudo ip addr add 10.55.99.7/26 dev ve-home-l2-br
#                ^^^^^^^^^^^^^
#      PE2: 10.55.99.8/26  PE3: 10.55.99.9/26
```

**What this does:** Configures the default namespace end of the home veth. This
is the machine network address — the IP that nodes use to communicate
with each other. The `/26` subnet (10.55.99.0/26) has 64 addresses, with .1 as
the anycast gateway and .7/.8/.9 as the three PE nodes. When this PE sends a
packet to 10.55.99.8 (PE2), it's on the same subnet, so it ARPs on
`ve-home-l2-br`. The ARP enters the bridge, which (via EVPN) knows PE2's MAC
is behind `vxlan0` → VTEP 10.55.96.154.

```bash
sudo sysctl -w \
  net.ipv6.conf.all.seg6_enabled=1 \
  net.ipv4.conf.all.forwarding=1 \
  net.ipv6.conf.all.forwarding=1 \
  net.ipv4.conf.all.rp_filter=0
```

**What this does:** Enables SRv6, IPv4/IPv6 forwarding, and disables RPF in
the default namespace. Even though the default namespace is primarily a "host"
(not a router), it needs forwarding enabled for some edge cases where traffic
transits through it, and SRv6/RPF settings for consistency.

### 1.16 Verify Phase 1

At this point, there are NO routing protocols. But local connectivity should
work:

```bash
# Verify namespaces exist
ip netns list
# Expected: l2-vxlan, pe-router

# Verify veth connectivity (default ↔ pe-router)
ping -c 1 -I ve-ovn-pe-cmn 10.55.96.21     # should work (local /30)
```

**What `-I ve-ovn-pe-cmn` does:** Forces the ping to use the specified source
interface. Without `-I`, the kernel might choose a different source interface,
and the reply might be undeliverable. With `-I`, we're explicitly testing the
veth pair.

```bash
# Verify l2-vxlan ↔ pe-router
sudo ip netns exec l2-vxlan ping -c 1 10.55.96.149   # should work
```

**What this tests:** The VTEP can reach the pe-router's CMN VRF gateway. This
is a critical local path — if this doesn't work, no VXLAN traffic can leave
the PE.

```bash
# Verify l2-vxlan has a default route
sudo ip netns exec l2-vxlan ip route show
# Expected: default via 10.55.96.149 dev ve-l2-pe-cmn
```

**What does NOT work yet:**
- No IS-IS → PEs can't reach each other's loopbacks or SRv6 locators
- No BGP → VRFs have no remote routes, only connected subnets
- No EVPN → VXLAN tunnels have no remote VTEPs in their FDB
- Pinging 10.55.99.8 from PE1 will fail (PE2 is unreachable)

---

## Phase 2 — IS-IS

**Goal:** Establish IS-IS adjacencies between PEs and the spine. After this phase,
every node can reach every other node's loopback and SRv6 locator prefix via
IPv6. This provides the underlay that all overlay protocols depend on.

**What IS-IS does for this solution:** IS-IS is the underlay routing protocol.
It discovers the topology (who is connected to whom), computes shortest paths
(using Dijkstra's algorithm), and installs IPv6 routes in the kernel. These
routes enable two critical things: (1) BGP sessions between PE loopbacks, and
(2) SRv6 packet forwarding between PEs via the locator prefixes.

### 2.1 Write the FRR Configs

FRR (Free Range Routing) is the routing suite that runs IS-IS, BGP, and other
protocols. It runs inside a Podman container that shares the pe-router
namespace's network stack. We need three files: `vtysh.conf` (shell config),
`daemons` (which daemons to start), and `frr.conf` (the routing config).

Create the config directories and files on each PE.

**On PE1:**

```bash
sudo mkdir -p /etc/frr/pe-router
echo "service integrated-vtysh-config" | sudo tee /etc/frr/pe-router/vtysh.conf
```

**What this does:** Creates the config directory and writes `vtysh.conf`. The
`service integrated-vtysh-config` line tells FRR's CLI shell (vtysh) to write
all daemon configs to a single `frr.conf` file instead of separate per-daemon
files. This simplifies management.

Create the daemons file (tells FRR which daemons to start):

```bash
sudo tee /etc/frr/pe-router/daemons << 'EOF'
zebra=yes
bgpd=yes
ospfd=no
ospf6d=no
ripd=no
ripngd=no
isisd=yes
pimd=no
pim6d=no
ldpd=no
nhrpd=no
eigrpd=no
babeld=no
sharpd=no
pbrd=no
bfdd=yes
fabricd=no
vrrpd=no
pathd=yes
staticd=yes
vtysh_enable=yes
zebra_options="  -A 127.0.0.1 -s 90000000"
bgpd_options="   -A 127.0.0.1"
isisd_options="  -A 127.0.0.1"
staticd_options="-A 127.0.0.1"
bfdd_options="   -A 127.0.0.1"
pathd_options="  -A 127.0.0.1"
EOF
```

**What this does:** Tells FRR which daemons to run and their startup options.

**Key daemons enabled and their roles:**
- `zebra=yes` — The core daemon. Zebra manages the kernel routing table (FIB),
  communicates with all other FRR daemons, and handles interface management.
  Every FRR deployment needs zebra. Think of it as the "operating system" for
  FRR — other daemons compute routes, but zebra installs them in the kernel.
- `bgpd=yes` — The BGP daemon. Handles BGP L3VPN sessions with the spine.
- `isisd=yes` — The IS-IS daemon. Handles IS-IS adjacencies and SPF.
- `staticd=yes` — Static route daemon. Manages static routes configured in FRR.
- `bfdd=yes` — BFD (Bidirectional Forwarding Detection) daemon. Provides
  sub-second failure detection for routing adjacencies. When a link fails, BFD
  detects it faster than IS-IS Hellos (milliseconds vs seconds).
- `pathd=yes` — Path computation daemon. Supports SRv6 path computation.

**Daemon options explained:**
- `-A 127.0.0.1` — Each daemon listens for vtysh connections on 127.0.0.1
  (localhost only). This is a security measure — you can only manage FRR from
  within the container.
- `-s 90000000` — (zebra only) Sets the internal netlink socket buffer to 90MB.
  The default is much smaller. With many routes (from BGP VPN) and many
  interfaces (veths, VRFs), the kernel → zebra communication can be bursty.
  A larger buffer prevents dropped messages during route installation storms.

Write `frr.conf` with IS-IS only (we'll add BGP and SRv6 in later phases):

```bash
sudo tee /etc/frr/pe-router/frr.conf << 'EOF'
frr defaults traditional
hostname pe1-router
log file /var/log/frr/frr.log
!
interface eth1
 ip router isis SRV6-LAB
 ipv6 router isis SRV6-LAB
 isis network point-to-point
exit
!
interface lo
 ip router isis SRV6-LAB
 ipv6 address fc00:0:0:1::16/128
 ipv6 router isis SRV6-LAB
 isis passive
exit
!
router isis SRV6-LAB
 is-type level-1
 net 49.0000.1000.f006.0016.00
 topology ipv6-unicast
 log-adjacency-changes
exit
!
end
EOF
```

**Line-by-line explanation of frr.conf:**

`frr defaults traditional` — Use traditional FRR defaults (timer values,
administrative distances, etc.). The alternative is `datacenter`, which uses
more aggressive timers. `traditional` is safer for a learning environment.

`hostname pe1-router` — Sets the hostname displayed in the vtysh prompt. Purely
cosmetic but helps identify which router you're connected to when managing
multiple nodes.

`log file /var/log/frr/frr.log` — Logs all FRR messages to this file inside
the container. Essential for troubleshooting. Check this log when adjacencies
don't form or routes are missing.

**Interface eth1 block:**

`ip router isis SRV6-LAB` — Enables IS-IS for **IPv4** on this interface.
`SRV6-LAB` is the IS-IS process name (local to this router — it doesn't need
to match on other routers). This tells isisd to send IS-IS Hello packets on
eth1 and consider it for adjacency formation.

`ipv6 router isis SRV6-LAB` — Enables IS-IS for **IPv6** on this interface.
IS-IS can carry both IPv4 and IPv6 topologies simultaneously. We need IPv6
because the loopbacks and SRv6 locators are IPv6 addresses.

`isis network point-to-point` — Tells IS-IS this is a point-to-point link
(exactly two routers). Without this, IS-IS assumes the link is a broadcast
network (like a shared Ethernet segment with many routers) and elects a DIS
(Designated Intermediate System). DIS election adds delay and complexity.
Since each PE ↔ spine link has exactly two endpoints, point-to-point is correct
and faster.

**Interface lo (loopback) block:**

`ip router isis SRV6-LAB` — Adds the loopback to IS-IS for IPv4.

`ipv6 address fc00:0:0:1::16/128` — Assigns the IPv6 loopback address (also
done at the kernel level earlier, but FRR needs to know about it too for proper
advertisement).

`ipv6 router isis SRV6-LAB` — Adds the loopback to IS-IS for IPv6. IS-IS will
advertise this /128 address in its LSPs so all other routers know how to reach
it.

`isis passive` — Marks the loopback as **passive**. A passive interface is
advertised in IS-IS (so others can route to it) but does NOT send or receive
IS-IS Hello packets. This makes sense for loopbacks — there's no adjacent
router on the loopback interface to form an adjacency with. You want to
advertise the address but not try to discover neighbors on it.

**Router isis block:**

`is-type level-1` — This router operates as Level-1 only (intra-area). It will
only form Level-1 adjacencies and maintain a Level-1 link-state database. In
a larger network, you'd use Level-1-2 or Level-2 for inter-area routing, but
this lab has a single area (49.0000).

`net 49.0000.1000.f006.0016.00` — The Network Entity Title (IS-IS's unique
identifier for this router). Breakdown:
- `49` — AFI (Address Family Indicator). 49 means private/local addressing.
- `0000` — Area ID. All nodes in this lab use area `0000` within AFI 49,
  forming the full area address `49.0000`. Routers must share the same area
  to form Level-1 adjacencies.
- `1000.f006.0016` — System ID (6 bytes, unique per router). This is
  derived from the router's identity — PE1 uses `0016`, PE2 uses `0017`, PE3
  uses `0018`, the spine uses `0001`.
- `00` — SEL (Selector). Always `00` for a router (non-zero values indicate
  specific services, but `00` is always used in practice).

`topology ipv6-unicast` — Enables the IPv6 unicast topology in IS-IS. By
default, IS-IS only carries IPv4 topology information. This command adds IPv6
support, causing IS-IS to compute separate shortest paths for IPv6 destinations
and install IPv6 routes. Without this, IS-IS would not advertise or compute
paths for IPv6 prefixes (loopbacks, SRv6 locators).

`log-adjacency-changes` — Logs a message whenever an IS-IS adjacency state
changes (Up, Down, Initializing). Invaluable for troubleshooting — if an
adjacency flaps, you'll see it in the log.

**On PE2:** Same structure, change:
- hostname → `pe2-router`
- loopback → `fc00:0:0:1::17/128`
- NET → `49.0000.1000.f006.0017.00`

**On PE3:** Same structure, change:
- hostname → `pe3-router`
- loopback → `fc00:0:0:1::18/128`
- NET → `49.0000.1000.f006.0018.00`

### 2.2 Start the pe-router FRR Container

```bash
sudo podman run -d --name frr-pe-router \
  --network=ns:/run/netns/pe-router \
  --privileged \
  -v /etc/frr/pe-router:/etc/frr \
  quay.io/frrouting/frr:10.5.2
```

**Flag-by-flag explanation:**

| Flag | What It Does | Why |
|------|-------------|-----|
| `-d` | Runs the container in detached (background) mode | We don't need an interactive terminal — FRR runs as a set of daemons |
| `--name frr-pe-router` | Names the container `frr-pe-router` | So we can reference it by name in `podman exec` commands instead of by container ID |
| `--network=ns:/run/netns/pe-router` | Shares the pe-router network namespace with the container | This is the KEY flag. Instead of giving the container its own network namespace (the default), it shares the pe-router namespace we created earlier. FRR's daemons see eth1, lo, VRFs, and all veth interfaces as if they were running directly in the pe-router namespace. |
| `--privileged` | Runs the container with full root capabilities | FRR needs to modify the kernel routing table (add/remove routes), manage interfaces, and set sysctls. These operations require elevated privileges. Without `--privileged`, FRR would fail with permission errors when trying to install routes. |
| `-v /etc/frr/pe-router:/etc/frr` | Bind-mounts the host directory to /etc/frr inside the container | FRR reads its config from /etc/frr. By mounting our config directory, FRR uses the frr.conf, daemons, and vtysh.conf we created. Changes to these files on the host are immediately visible inside the container. |
| `quay.io/frrouting/frr:10.5.2` | The FRR container image | Version 10.5.2 of FRR, which supports IS-IS, BGP, SRv6, and EVPN |

### 2.3 Configure the Spine

On the containerlab host, the spine already has its config bind-mounted. The
IS-IS portion of the spine's `frr.conf`:

```
interface lo
 ip address 1.1.1.1/32
 ipv6 address fc00:0:0:1::1/128
 ip router isis UNDERLAY
 ipv6 router isis UNDERLAY
 isis passive

interface eth1
 ip router isis UNDERLAY
 ipv6 router isis UNDERLAY
 isis network point-to-point

interface eth2
 ip router isis UNDERLAY
 ipv6 router isis UNDERLAY
 isis network point-to-point

interface eth3
 ip router isis UNDERLAY
 ipv6 router isis UNDERLAY
 isis network point-to-point

router isis UNDERLAY
 is-type level-1
 net 49.0000.0000.0000.0001.00
 topology ipv6-unicast
 log-adjacency-changes
```

**Spine-specific details:**

- `ip address 1.1.1.1/32` — The spine's IPv4 loopback (used as BGP router-id)
- Three `interface` blocks (eth1, eth2, eth3) — one per PE connection. All are
  IS-IS enabled with point-to-point mode.
- `net 49.0000.0000.0000.0001.00` — The spine's NET with system ID `0000.0000.0001`
  and the same area `49.0000` as the PEs.

**Note:** The spine uses process name `UNDERLAY`, PEs use `SRV6-LAB`. This doesn't
matter for adjacency — IS-IS process names are purely local identifiers. What
matters for adjacency is: (1) both routers are in the same area (`49.0000`),
(2) both are the same level (Level-1), and (3) IS-IS is enabled on the
connecting interfaces. Process names are just used internally by FRR to
distinguish between multiple IS-IS instances on the same router (rare).

### 2.4 Verify IS-IS

Wait 10-15 seconds for adjacencies to form, then:

```bash
# On the spine:
sudo podman exec clab-srv6-lab-spine vtysh -c 'show isis neighbor'
# Expected: three neighbors, all State = Up
```

**What `podman exec clab-srv6-lab-spine vtysh -c '...'` does:** Runs a single
vtysh command inside the spine container. `vtysh` is FRR's command-line interface
(similar to Cisco IOS CLI). The `-c` flag runs a single command and exits.

```bash
# On PE1:
sudo podman exec frr-pe-router vtysh -c 'show isis neighbor'
# Expected: one neighbor (spine), State = Up
```

Check that IS-IS installed routes into the kernel:

```bash
# On the spine — can it reach PE loopbacks?
sudo podman exec clab-srv6-lab-spine vtysh -c 'show ipv6 route isis'
# Expected:
# I>* fc00:0:0:1::16/128 via fe80::..., eth1
# I>* fc00:0:0:1::17/128 via fe80::..., eth2
# I>* fc00:0:0:1::18/128 via fe80::..., eth3
```

**What to verify:** Three loopback routes, one per PE, all marked with `I>*`
(IS-IS, best route, installed in kernel). The `via fe80::` next-hops are IPv6
link-local addresses — IS-IS on point-to-point links uses these automatically.

**Troubleshooting:**

| Problem | Check |
|---------|-------|
| No adjacency | Is `eth1` in pe-router ns? `sudo ip netns exec pe-router ip link show eth1` |
| Stuck in Init | Area mismatch — verify both are `49.0000` and Level-1 |
| Route not installed | `show isis database detail` — check LSP is being generated |
| "No IS-IS process" | Check `daemons` file — is `isisd=yes`? |

---

## Phase 3 — SRv6

**Goal:** Enable SRv6 on all nodes. IS-IS will advertise each node's locator
prefix, making the SRv6 data plane functional. After this phase, each node's
SRv6 locator (/48) is reachable from every other node, and the kernel has SID
entries for local SRv6 behaviors.

**What SRv6 does for this solution:** SRv6 provides the transport encapsulation
for VRF traffic. When a PE needs to send a CMN VRF packet to another PE, it
wraps the packet in an IPv6 header with the destination set to the remote PE's
SRv6 SID. Transit routers (the spine) just forward it as normal IPv6. The remote PE
recognizes the SID and delivers the inner packet to the correct VRF. SRv6
replaces MPLS as the transport mechanism — it's simpler because transit routers
don't need any special configuration.

### 3.1 Add SRv6 to PE's FRR Config

Add these sections to each PE's `/etc/frr/pe-router/frr.conf`:

**On PE1:**

```
router isis SRV6-LAB
 segment-routing on
 segment-routing srv6
  locator MAIN
 exit
```

**Line-by-line:**

`segment-routing on` — Enables Segment Routing extensions in IS-IS. This tells
IS-IS to include Segment Routing TLVs in its LSPs (Link-State PDUs). Other
routers will see these TLVs and know that this router supports SRv6.

`segment-routing srv6` — Enters the SRv6-specific configuration sub-mode under
IS-IS. (As opposed to SR-MPLS, which would use `segment-routing mpls`.)

`locator MAIN` — Tells IS-IS to advertise the locator named `MAIN` (defined
below) in its LSPs. This is how other routers learn that PE1 owns the
`fd00:30:16::/48` prefix. IS-IS floods this information to all routers in the
area, and they install routes to reach this prefix.

```
segment-routing
 srv6
  encapsulation
   source-address fd00:30:16::
  locators
   locator MAIN
    prefix fd00:30:16::/48 block-len 32 node-len 16
    behavior usid
   exit
  exit
 exit
```

**Line-by-line:**

`segment-routing` — Top-level SRv6 configuration block (shared across all
protocols, not specific to IS-IS).

`srv6` → `encapsulation` → `source-address fd00:30:16::` — Sets the IPv6
source address used in ALL SRv6-encapsulated packets from this node. When PE1
wraps a VRF packet in SRv6, the outer IPv6 header has:
- src = `fd00:30:16::` (this setting)
- dst = the remote SID (e.g., `fd00:30:17:3::`)

Using the locator prefix as the source (instead of the loopback) means transit
routers can identify the originating node from the source address, which helps
with troubleshooting and ECMP.

`locators` → `locator MAIN` — Defines a locator named `MAIN`. The name is
arbitrary — it's referenced by IS-IS and BGP when they need to know which
locator to use.

`prefix fd00:30:16::/48` — The SRv6 locator prefix. This /48 block "belongs"
to PE1. All SRv6 SIDs for PE1 are addresses within this range. Transit routers
only need a single /48 route to reach all of PE1's SIDs.

`block-len 32` — The first 32 bits (`fd00:0030`) identify the SRv6 domain. All
nodes in this fabric share this block prefix.

`node-len 16` — The next 16 bits (`0016`) identify PE1 specifically. Combined
with the block, `fd00:0030:0016` uniquely identifies PE1.

`behavior usid` — Enables micro-SID (uSID) mode. Instead of putting multiple
full 128-bit SIDs in a Segment Routing Header (SRH), micro-SID packs short
(16-bit) instructions into the destination address itself. For single-segment
paths (which is most traffic in this star topology), this means NO SRH is
needed at all — the destination address alone encodes both the target node
and the action. This reduces overhead compared to traditional SRv6.

**On PE2:** Change source-address to `fd00:30:17::` and prefix to `fd00:30:17::/48`.
**On PE3:** Change source-address to `fd00:30:18::` and prefix to `fd00:30:18::/48`.

Apply by restarting the container:

```bash
sudo podman restart frr-pe-router
```

**What this does:** Stops and restarts the FRR container. On restart, FRR
re-reads `frr.conf` and applies the new SRv6 configuration. The IS-IS
adjacency will briefly drop (you'll see "Adjacency Down" in the logs) and
re-form within ~10 seconds.

Or apply live via vtysh (no restart needed):

```bash
sudo podman exec frr-pe-router vtysh
# Then paste the config blocks above
```

**What this does:** Opens an interactive vtysh session where you can paste
configuration directly. Changes take effect immediately and the adjacency stays
up. Preferred for production; restart is simpler for learning.

### 3.2 Spine SRv6 Config

The spine's config already includes SRv6 (from the initial setup):

```
router isis UNDERLAY
 segment-routing on
 segment-routing srv6
  locator MAIN

segment-routing
 srv6
  encapsulation
   source-address fc00:0:0:1::1
  locators
   locator MAIN
    prefix fd00:30:1::/48 block-len 32 node-len 16
    behavior usid
```

The spine uses `fc00:0:0:1::1` as its SRv6 source address and `fd00:30:1::/48` as its
locator. Note the spine's locator uses node-id `0001` (reflected in `fd00:30:0001`),
while PEs use `0016`, `0017`, `0018`.

### 3.3 Verify SRv6

```bash
# Check locator is Up:
sudo podman exec frr-pe-router vtysh -c 'show segment-routing srv6 locator'
# Expected: MAIN  1  fd00:30:16::/48  Up
```

**What "Up" means:** FRR has successfully registered the locator with the
kernel and IS-IS is advertising it. If it shows "Down," check that IS-IS is
running and the `segment-routing` config is correct.

```bash
# Check IS-IS now advertises locator prefixes:
sudo podman exec clab-srv6-lab-spine vtysh -c 'show ipv6 route isis'
# Expected (new entries):
# I>* fd00:30:16::/48 via fe80::..., eth1
# I>* fd00:30:17::/48 via fe80::..., eth2
# I>* fd00:30:18::/48 via fe80::..., eth3
```

**What changed:** Before SRv6, IS-IS only advertised loopback /128 addresses.
Now it ALSO advertises /48 locator prefixes. These /48 routes are what transit
routers use to forward SRv6 packets. When the spine sees a packet destined for
`fd00:30:17:3::`, it matches against `fd00:30:17::/48` and forwards via eth2
toward PE2.

```bash
# Check SRv6 SID table in the kernel:
sudo ip netns exec pe-router ip -6 route show | grep seg6local
# Expected: fd00:30:16:X:: entries with seg6local actions
```

**What you'll see:** The kernel has routes like:
```
fd00:30:16:3:: encap seg6local action End.DT4 vrftable 4 dev CMN
```

This is the SRv6 SID route. It tells the kernel: "when a packet arrives for
`fd00:30:16:3::`, perform End.DT4 (decapsulate the outer IPv6 header and route
the inner IPv4 packet using VRF table 4/CMN)." FRR's zebra daemon programmed
this route when BGP allocated the VRF SID. The `seg6local` keyword is the
kernel's representation of an SRv6 local SID behavior.

---

## Phase 4 — BGP L3VPN

**Goal:** Exchange VRF routes between PEs via BGP. After this phase, each
PE's CMN VRF will have routes to every other PE's CMN subnets, with SRv6
encapsulation instructions. VTEP reachability will be established.

**What BGP L3VPN does for this solution:** BGP L3VPN is the control plane that
distributes VRF routing information between PEs. It tells each PE: "PE2's CMN
VRF has subnet 10.55.96.152/30, and to reach it, SRv6-encapsulate to
fd00:30:17:3::." Without this, each PE only knows about its own directly
connected subnets. With it, every PE has a complete routing table for the CMN
VRF spanning all PEs. Critically, this enables VTEP reachability — the l2-vxlan
namespaces can reach each other through the CMN VRF routes that BGP distributes.

### 4.1 Add VRF and BGP Config to PE

Add these sections to each PE's `/etc/frr/pe-router/frr.conf`:

**On PE1:**

```
vrf CMN
 ip route 10.55.99.0/26 10.55.96.150
exit-vrf
```

**What this does:** Adds a static route inside the CMN VRF. This route says:
"the machine network (10.55.99.0/26) is reachable via 10.55.96.150." The
next-hop `10.55.96.150` is PE1's VTEP IP (in the l2-vxlan namespace).

**Why this route exists:** The machine network lives behind the l2-vxlan
namespace's VXLAN bridge. The CMN VRF needs to know how to reach it. This
static route provides that information, and `redistribute static` (below)
includes it in BGP, so other PEs also learn about PE1's machine network
through the CMN VRF.

```
vrf EDN
exit-vrf
```

**What this does:** Defines the EDN VRF block (currently empty). The VRF was
created at the kernel level earlier; this tells FRR about it so BGP can
reference it.

```
router bgp 65500
 bgp router-id 172.31.250.106
 no bgp default ipv4-unicast
 bgp default ipv6-unicast
 no bgp network import-check
 bgp allow-martian-nexthop
 neighbor SPINE peer-group
 neighbor SPINE remote-as 65500
 neighbor SPINE capability dynamic
 neighbor SPINE capability extended-nexthop
 neighbor fc00:0:0:1::1 peer-group SPINE
```

**Line-by-line:**

`router bgp 65500` — Creates the BGP process with ASN 65500. All nodes in this
L3VPN domain share this AS number (making it iBGP). The ASN is a 16-bit number
(65500 is in the private range 64512-65534).

`bgp router-id 172.31.250.106` — A 32-bit unique identifier for this BGP
speaker. It appears in BGP messages and MUST be unique per router. Using the
management IP is a common convention. PE2 uses `.107`, PE3 uses `.108`.

`no bgp default ipv4-unicast` — By default, BGP auto-activates the IPv4
unicast address family for every neighbor. This command disables that. We want
explicit control over which address families each neighbor uses. Without this,
every neighbor would automatically exchange IPv4 unicast routes, which we don't
want — we only want IPv4 VPN and IPv6 VPN.

`bgp default ipv6-unicast` — Auto-activates IPv6 unicast for every neighbor.
We peer over IPv6 loopback addresses, so we need basic IPv6 reachability.

`no bgp network import-check` — Disables the check that verifies a network
exists in the RIB before advertising it via BGP. Without this, if a route is
briefly missing (e.g., during a kernel route update), BGP might withdraw it.
Disabling the check prevents unnecessary route churn.

`bgp allow-martian-nexthop` — Allows BGP to accept next-hops that are
"martian" addresses (addresses that would normally be rejected because they're
not globally routable, like private addresses). SRv6 SIDs use `fd00::/8` (ULA
space), which BGP would normally reject as a next-hop. This command permits it.

`neighbor SPINE peer-group` — Creates a peer-group template called `SPINE`. Any
configuration applied to the peer-group applies to all members. This avoids
repeating settings for each neighbor.

`neighbor SPINE remote-as 65500` — The SPINE peer group is in AS 65500 (same as us
= iBGP). If this were a different AS, it would be eBGP.

`neighbor SPINE capability dynamic` — Enables dynamic capability negotiation.
Allows the BGP session to add or remove address families without tearing down
and re-establishing the entire session. Useful when you add EVPN or VPN
families after the session is already established.

`neighbor SPINE capability extended-nexthop` — Enables RFC 8950 extended next-hop
encoding. This allows IPv4 VPN routes to be advertised with IPv6 next-hops.
This is essential because we peer over IPv6 (loopback `fc00::` addresses) but
the VPN routes are IPv4. Without this, BGP couldn't encode the IPv6 next-hop
for IPv4 VPN prefixes, and the session would fail.

`neighbor fc00:0:0:1::1 peer-group SPINE` — Creates a BGP session to the spine's
loopback and assigns it to the SPINE peer-group. The session rides over the IS-IS
underlay — IS-IS ensures `fc00:0:0:1::1` (the spine's loopback) is reachable from
PE1's loopback.

```
 segment-routing srv6
  locator MAIN
 exit
 sid vpn per-vrf export auto
```

`segment-routing srv6` → `locator MAIN` — Tells BGP to use the MAIN SRv6
locator (defined earlier) when allocating SIDs for VPN services. When BGP needs
an SRv6 SID for a VRF, it allocates one from the MAIN locator's address range.

`sid vpn per-vrf export auto` — Tells BGP to automatically allocate one SRv6
SID per VRF. "Per-VRF" means all routes in a VRF share the same SID (as opposed
to "per-CE" which would allocate a SID per customer edge prefix). "Auto" means
FRR chooses the function ID automatically. The result is something like
`fd00:30:16:3::` for CMN VRF — FRR picked function 3.

```
 address-family ipv4 unicast
  redistribute static
  neighbor SPINE next-hop-self
 exit-address-family
```

`address-family ipv4 unicast` — Enters IPv4 unicast configuration. This
configures how PE1 handles IPv4 routes in the default (non-VRF) context.

`redistribute static` — Injects static routes into BGP. This includes the
machine network route (10.55.99.0/26 via 10.55.96.150) that we defined in the
VRF block.

`neighbor SPINE next-hop-self` — When advertising routes to the spine, replace the
next-hop with PE1's own address. By default, iBGP preserves the original
next-hop. `next-hop-self` ensures the spine always sees PE1 as the next-hop,
simplifying routing decisions.

```
 address-family ipv4 vpn
  neighbor SPINE activate
  neighbor SPINE next-hop-self
 exit-address-family
```

`address-family ipv4 vpn` — Activates the IPv4 VPN (VPNv4) address family
with the spine. This is where L3VPN routes are exchanged. VPN routes carry
additional metadata (RD, RT, SRv6 SID) that standard IPv4 routes don't.

`neighbor SPINE activate` — Enables the IPv4 VPN family for the spine peer. Without
this, the BGP session would be established but wouldn't exchange any VPN
routes.

```
 address-family ipv6 vpn
  neighbor SPINE activate
 exit-address-family
```

Same as above but for IPv6 VPN routes. Enabled for completeness, though most
VRF traffic in this lab is IPv4.

```
router bgp 65500 vrf EDN
 sid vpn per-vrf export auto
 !
 address-family ipv4 unicast
  redistribute connected
  redistribute static
  rd vpn export 172.31.250.106:2
  rt vpn both 65004:2
  export vpn
  import vpn
 exit-address-family
exit
```

`router bgp 65500 vrf EDN` — Configures BGP for the EDN VRF. This is a
separate BGP instance within the same AS (65500) but scoped to the EDN VRF.

`sid vpn per-vrf export auto` — Allocates an SRv6 SID for EDN (separate from
CMN's SID).

`redistribute connected` — Injects connected routes (directly attached
subnets) into BGP for this VRF.

`redistribute static` — Injects static routes into BGP for this VRF.

`rd vpn export 172.31.250.106:2` — Sets the Route Distinguisher. The `:2`
suffix matches the EDN VRF table number. The full RD
(`172.31.250.106:2`) makes this PE's EDN routes unique globally.

`rt vpn both 65004:2` — Sets the Route Target for both import and export.
Routes leaving EDN carry RT `65004:2`. Routes arriving with RT `65004:2`
are imported into EDN. This is how VRF membership works across PEs.

`export vpn` — Takes EDN's routes and pushes them into the VPN address family
(with RD and RT attached). The spine then reflects them to other PEs.

`import vpn` — Takes incoming VPN routes (from other PEs, reflected by the spine)
and installs them in the EDN VRF if their RT matches.

```
router bgp 65500 vrf CMN
 sid vpn per-vrf export auto
 !
 address-family ipv4 unicast
  redistribute kernel
  redistribute connected
  redistribute static
  rd vpn export 172.31.250.106:4
  rt vpn both 65004:4
  export vpn
  import vpn
 exit-address-family
exit
```

Same pattern as EDN but for CMN VRF. Notable difference:

`redistribute kernel` — Includes kernel-installed routes (routes added by
zebra from other protocols or by manual `ip route add`). This catches routes
that might be added by other parts of the system.

`rd vpn export 172.31.250.106:4` — RD with `:4` suffix for CMN. Combined
with the router-id prefix, this uniquely identifies PE1's CMN VRF routes.

`rt vpn both 65004:4` — RT for CMN. All PEs use the same RT for their CMN
VRFs, so routes are shared between all CMN instances.

**On PE2:** Change router-id to `172.31.250.107`, RDs to `172.31.250.107:X`,
static route next-hop to `10.55.96.154`.

**On PE3:** Change router-id to `172.31.250.108`, RDs to `172.31.250.108:X`,
static route next-hop to `10.55.96.158`.

### 4.2 Spine BGP Config

The spine's BGP (already in its frr.conf):

```
router bgp 65500
 bgp router-id 1.1.1.1
 no bgp default ipv4-unicast
 bgp default ipv6-unicast
 no bgp network import-check
 bgp allow-martian-nexthop
 !
 neighbor PE peer-group
 neighbor PE remote-as 65500
 neighbor PE capability dynamic
 neighbor PE capability extended-nexthop
 neighbor PE update-source lo
 neighbor fc00:0:0:1::16 peer-group PE
 neighbor fc00:0:0:1::17 peer-group PE
 neighbor fc00:0:0:1::18 peer-group PE
 !
 address-family ipv4 vpn
  neighbor PE activate
  neighbor PE route-reflector-client
 exit-address-family
 !
 address-family ipv6 vpn
  neighbor PE activate
  neighbor PE route-reflector-client
 exit-address-family
```

**Key spine-specific lines:**

`neighbor PE update-source lo` — The spine uses its loopback as the source for all
BGP sessions. This means BGP packets from the spine have source IP `fc00:0:0:1::1`
(the loopback address), not any interface-specific address. Using a loopback
ensures the session stays up even if a specific interface flaps (as long as
some path to the spine exists). On the PEs, the sessions are established TO
`fc00:0:0:1::1`, matching this update-source.

`neighbor PE route-reflector-client` — This is the route reflector
configuration. It tells the spine: "when you receive a VPN route from one PE, re-
advertise (reflect) it to all other PEs in the `PE` peer-group." Without this,
iBGP's default behavior prevents the spine from re-advertising routes learned from one
iBGP peer to another iBGP peer. With route reflection, the spine acts as a central
hub that distributes VPN routes between all PEs.

Three `neighbor` statements (one per PE loopback) — the spine peers with all three
PEs. When PE1 sends a CMN route, the spine reflects it to PE2 and PE3.

### 4.3 Apply and Verify

```bash
sudo podman restart frr-pe-router
# Wait 15 seconds for BGP to establish
```

```bash
# Check BGP sessions on the spine:
sudo podman exec clab-srv6-lab-spine vtysh -c 'show bgp ipv4 vpn summary'
# Expected: 3 peers, each with PfxRcd = 5
```

**What PfxRcd = 5 means:** Each PE exports approximately 5 routes from its CMN
VRF (connected subnets for the veths, the loopback, and the static route).
If PfxRcd shows 0 or the state shows `Active`/`Connect` instead of a number,
the session isn't exchanging routes — check the BGP configuration and IS-IS
reachability.

```bash
# Check CMN VRF on PE1 — are remote routes there?
sudo podman exec frr-pe-router vtysh -c 'show ip route vrf CMN'
# Look for B> entries with seg6 next-hops:
#   B>  10.55.96.152/30 ... seg6 fd00:30:17:3::  ← PE2's VTEP subnet
#   B>  10.55.96.156/30 ... seg6 fd00:30:18:3::  ← PE3's VTEP subnet
```

`B>` means BGP-learned, best route. `seg6 fd00:30:17:3::` means "to reach this
prefix, SRv6-encapsulate the packet with destination `fd00:30:17:3::`." These
routes were learned from PE2 and PE3 via the spine's route reflection.

**Test VTEP reachability:**

```bash
# From PE1's l2-vxlan namespace, ping PE2's VTEP:
sudo ip netns exec l2-vxlan ping -c 3 10.55.96.154
# This should WORK — traffic goes:
#   l2-vxlan → ve-l2-pe-cmn → CMN VRF → SRv6 → spine → PE2 → CMN VRF → l2-vxlan
```

**This is the critical checkpoint.** If this ping works, the entire L3 path is
operational: namespace crossing, VRF routing, SRv6 encapsulation, IS-IS
underlay forwarding, SRv6 decapsulation, and namespace delivery. Every layer
from Phase 1-4 is functioning. If it doesn't work, debug bottom-up: IS-IS
adjacencies, then SRv6 locator routes, then BGP sessions, then VPN routes.

---

## Phase 5 — BGP EVPN + VXLAN

**Goal:** Establish BGP EVPN sessions between l2-vxlan FRR instances. After
this phase, VXLAN tunnels form and the machine network is connected end-to-end.

**What EVPN + VXLAN does for this solution:** EVPN provides the control plane
for the Layer 2 overlay. It distributes MAC/IP information between VTEPs so
they know where to send VXLAN-encapsulated frames. VXLAN provides the data
plane — the actual tunneling of Ethernet frames between VTEPs. Together, they
create a virtual Ethernet switch spanning all three PEs, giving the nodes the
Layer 2 connectivity it needs for the machine network.

### 5.1 Write the l2-vxlan FRR Config

**On PE1 (route reflector):**

```bash
sudo mkdir -p /etc/frr/l2-vxlan
echo "service integrated-vtysh-config" | sudo tee /etc/frr/l2-vxlan/vtysh.conf

sudo tee /etc/frr/l2-vxlan/daemons << 'EOF'
zebra=yes
bgpd=yes
ospfd=no
ospf6d=no
ripd=no
ripngd=no
isisd=no
pimd=no
pim6d=no
ldpd=no
nhrpd=no
eigrpd=no
babeld=no
sharpd=no
pbrd=no
bfdd=no
fabricd=no
vrrpd=no
pathd=no
staticd=yes
vtysh_enable=yes
zebra_options="  -A 127.0.0.1 -s 90000000"
bgpd_options="   -A 127.0.0.1"
staticd_options="-A 127.0.0.1"
EOF
```

**Why fewer daemons than pe-router?** The l2-vxlan namespace only needs:
- `zebra` — for kernel route/FDB management (installs EVPN-learned MAC entries
  into the bridge FDB)
- `bgpd` — for EVPN BGP sessions
- `staticd` — for static routes

No `isisd` (IS-IS runs in pe-router), no `bfdd` (BFD runs in pe-router), no
`pathd` (no SRv6 path computation here). Each FRR instance is minimal — only
the daemons needed for that namespace's role.

```bash
sudo tee /etc/frr/l2-vxlan/frr.conf << 'EOF'
frr defaults traditional
hostname pe1-l2vxlan
log file /var/log/frr/frr.log
!
router bgp 65501
 bgp router-id 1.1.1.6
 no bgp default ipv4-unicast
 bgp default ipv6-unicast
 bgp cluster-id 2.2.2.2
 no bgp network import-check
 bgp allow-martian-nexthop
 neighbor EVPN-PEERS peer-group
 neighbor EVPN-PEERS remote-as 65501
 neighbor EVPN-PEERS update-source 10.55.96.150
 neighbor 10.55.96.154 peer-group EVPN-PEERS
 neighbor 10.55.96.158 peer-group EVPN-PEERS
 !
 address-family ipv4 unicast
  redistribute connected
  redistribute static
  redistribute kernel
  neighbor EVPN-PEERS activate
  neighbor EVPN-PEERS route-reflector-client
 exit-address-family
 !
 address-family l2vpn evpn
  neighbor EVPN-PEERS activate
  neighbor EVPN-PEERS route-reflector-client
  advertise-all-vni
  advertise-svi-ip
 exit-address-family
exit
!
end
EOF
```

**Line-by-line explanation:**

`router bgp 65501` — Creates the EVPN BGP process with ASN 65501. This is a
DIFFERENT AS than the L3VPN BGP (65500). The two BGP instances are completely
separate — different AS numbers, different namespaces, different FRR processes.
They don't directly interact. AS 65501 is used for EVPN route exchange between
l2-vxlan instances.

`bgp router-id 1.1.1.6` — Unique identifier for this BGP speaker. PE2 uses
`1.1.1.7`, PE3 uses `1.1.1.8`. These don't overlap with the L3VPN router-IDs
(`172.31.250.X`) because they're in a different BGP instance.

`bgp cluster-id 2.2.2.2` — Identifies this router as a **route reflector**.
The cluster-id prevents routing loops in route reflection: if a reflected route
returns to a router and the cluster-id is already in the route's cluster-list,
the route is rejected (loop detected). The value `2.2.2.2` is arbitrary but
must be unique per route reflector in the AS.

**Why PE1 is the EVPN RR:** The spine has no l2-vxlan namespace (it's a pure underlay
device), so it can't participate in EVPN. One of the PEs must serve as the RR.
PE1 (the first PE, designated "master-0" in the production deployment) takes
this role.

`neighbor EVPN-PEERS peer-group` — Peer group template for EVPN peers.

`neighbor EVPN-PEERS remote-as 65501` — iBGP (same AS).

`neighbor EVPN-PEERS update-source 10.55.96.150` — Use the VTEP IP as the
source for BGP sessions. These sessions run over the VTEP IPs (not IS-IS
loopbacks), because the l2-vxlan namespace only has connectivity through the
VTEP IP → CMN VRF → SRv6 path.

`neighbor 10.55.96.154 peer-group EVPN-PEERS` — Peer with PE2's VTEP.
`neighbor 10.55.96.158 peer-group EVPN-PEERS` — Peer with PE3's VTEP.

These addresses are reachable because BGP L3VPN (from Phase 4) distributes the
CMN VRF routes that include the VTEP subnets. The EVPN BGP sessions literally
ride on top of the L3VPN infrastructure.

`neighbor EVPN-PEERS route-reflector-client` — Reflect EVPN routes from PE2 to
PE3 and vice versa. PE2 and PE3 only peer with PE1 (the RR), not with each
other. PE1 ensures they each see the other's EVPN routes.

**L2VPN EVPN address family:**

`address-family l2vpn evpn` — Activates the L2VPN EVPN address family. This is
the heart of EVPN: it carries Type-2 (MAC/IP), Type-3 (multicast/BUM), and
other EVPN route types.

`advertise-all-vni` — Tells FRR's zebra to automatically discover all VNIs
on the system (by inspecting VXLAN devices) and advertise them via EVPN.
Zebra finds `vxlan0` with VNI 330, and BGP advertises a Type-3 (Inclusive
Multicast) route for VNI 330 with VTEP IP 10.55.96.150. This is how other
VTEPs learn that PE1 participates in VNI 330.

`advertise-svi-ip` — Includes the SVI (bridge VLAN interface) IP addresses in
EVPN Type-2 routes. The SVI IP here is the anycast gateway (10.55.99.1).
Including it enables ARP suppression — remote VTEPs can answer ARP requests for
the gateway locally instead of flooding them.

**On PE2 (client):**

```bash
sudo tee /etc/frr/l2-vxlan/frr.conf << 'EOF'
frr defaults traditional
hostname pe2-l2vxlan
log file /var/log/frr/frr.log
!
router bgp 65501
 bgp router-id 1.1.1.7
 no bgp default ipv4-unicast
 bgp default ipv6-unicast
 no bgp network import-check
 bgp allow-martian-nexthop
 neighbor EVPN-PEERS peer-group
 neighbor EVPN-PEERS remote-as 65501
 neighbor EVPN-PEERS update-source 10.55.96.154
 neighbor 10.55.96.150 peer-group EVPN-PEERS
 !
 address-family ipv4 unicast
  redistribute connected
  redistribute static
  redistribute kernel
  neighbor EVPN-PEERS activate
 exit-address-family
 !
 address-family l2vpn evpn
  neighbor EVPN-PEERS activate
  advertise-all-vni
  advertise-svi-ip
 exit-address-family
exit
!
end
EOF
```

**Key differences from PE1:**
- `bgp router-id 1.1.1.7` — unique ID for PE2
- `update-source 10.55.96.154` — PE2's VTEP IP
- `neighbor 10.55.96.150` — PE2 only peers with PE1 (the RR), NOT with PE3
- **No `cluster-id`** — PE2 is not a route reflector
- **No `route-reflector-client`** — PE2 doesn't reflect routes; it just sends
  and receives

**On PE3:** Same as PE2 but:
- hostname → `pe3-l2vxlan`
- router-id → `1.1.1.8`
- update-source → `10.55.96.158`
- neighbor → `10.55.96.150` (still PE1, the RR)

### 5.2 Start the l2-vxlan FRR Container

```bash
sudo podman run -d --name frr-l2-vxlan \
  --network=ns:/run/netns/l2-vxlan \
  --privileged \
  -v /etc/frr/l2-vxlan:/etc/frr \
  quay.io/frrouting/frr:10.5.2
```

**Same pattern as pe-router's FRR container, but:**
- `--name frr-l2-vxlan` — different container name
- `--network=ns:/run/netns/l2-vxlan` — shares the l2-vxlan namespace (not
  pe-router). This FRR instance sees `br0`, `vxlan0`, `ve-l2-pe-cmn`, and all
  other l2-vxlan interfaces. It does NOT see eth1, IS-IS, or the underlay.

### 5.3 Verify EVPN

```bash
# Check EVPN sessions (from PE1):
sudo podman exec frr-l2-vxlan vtysh -c 'show bgp l2vpn evpn summary'
# Expected: 2 peers (PE2, PE3), each with PfxRcd > 0
```

**What to check:** Both PE2 (10.55.96.154) and PE3 (10.55.96.158) should show
as Established with a non-zero prefix count. If the state shows `Active`, the
TCP connection is failing — check VTEP reachability (`sudo ip netns exec
l2-vxlan ping 10.55.96.154`). If the state shows `Established` but PfxRcd is
0, EVPN is running but no routes are being exchanged — check `advertise-all-vni`
and that `vxlan0` is up.

```bash
# Check EVPN route types:
sudo podman exec frr-l2-vxlan vtysh -c 'show bgp l2vpn evpn'
# Look for:
#   [2]:[0]:[48]:[...MAC...]  ← Type-2: learned MACs
#   [3]:[0]:[32]:[10.55.96.XXX]  ← Type-3: VTEP registration
```

**Type-2 routes** mean EVPN has learned specific MAC addresses from remote
PEs. **Type-3 routes** mean remote VTEPs have registered for BUM
(Broadcast/Unknown/Multicast) traffic on this VNI.

```bash
# Check VNI discovery:
sudo podman exec frr-l2-vxlan vtysh -c 'show evpn vni'
# Expected: VNI 330, # Remote VTEPs = 2
```

**What this shows:** Zebra discovered VNI 330 from the `vxlan0` device and
EVPN has learned about 2 remote VTEPs. If "Remote VTEPs = 0", Type-3 routes
aren't being received — check EVPN session status.

```bash
# Check kernel FDB (EVPN-installed entries):
sudo ip netns exec l2-vxlan bridge fdb show dev vxlan0
# Expected:
#   00:00:00:00:00:00 dst 10.55.96.154 self permanent   ← BUM entry for PE2
#   00:00:00:00:00:00 dst 10.55.96.158 self permanent   ← BUM entry for PE3
#   xx:xx:xx:xx:xx:xx dst 10.55.96.154 self extern_learn ← specific MACs
```

**What the FDB entries mean:**
- `00:00:00:00:00:00 dst 10.55.96.154 self permanent` — The all-zeros MAC is
  a wildcard BUM entry. All broadcast and unknown unicast frames are sent to
  VTEP 10.55.96.154 (PE2). Created by the EVPN Type-3 route. `permanent` means
  it persists until the EVPN session drops.
- `xx:xx:xx:xx:xx:xx dst 10.55.96.154 self extern_learn` — A specific MAC-to-
  VTEP mapping. Frames for this MAC go directly to PE2. Created by the EVPN
  Type-2 route. `extern_learn` means the entry was programmed by an external
  control plane (EVPN via zebra), not learned from data-plane flooding.

**Troubleshooting:**

| Problem | Check |
|---------|-------|
| EVPN session not establishing | `sudo ip netns exec l2-vxlan ping 10.55.96.154` — VTEP reachable? |
| No VNI in `show evpn vni` | Is vxlan0 up? `sudo ip netns exec l2-vxlan ip link show vxlan0` |
| No Type-3 routes | `advertise-all-vni` missing, or zebra can't see vxlan0 |
| MACs not learning | Is `ve-l2-br-home` in br0 with VLAN 30? `bridge vlan show` |
| FDB empty | Check EVPN routes first; if routes exist but FDB is empty, zebra may be failing to install — check zebra logs |

---

## Phase 6 — End-to-End Verification

Everything is configured. Let's verify the full stack, layer by layer.

### 6.1 The Ping Test

```bash
# From PE1's default namespace:
ping -c 3 10.55.99.8    # → PE2
ping -c 3 10.55.99.9    # → PE3

# From PE2:
ping -c 3 10.55.99.7    # → PE1
ping -c 3 10.55.99.9    # → PE3
```

If these work, every layer is functioning:
IS-IS → SRv6 → BGP L3VPN → VTEP reachability → BGP EVPN → VXLAN → bridge → host.

A successful ping means: the host resolved the destination MAC via ARP (L2),
the bridge forwarded the frame through VXLAN (L2 overlay), the VTEP sent the
VXLAN packet through the CMN VRF (L3 VPN), the CMN VRF SRv6-encapsulated it
(transport), the underlay routed the SRv6 packet to the remote PE (IS-IS),
the remote PE decapsulated (SRv6 → VRF → VTEP → bridge → host), and the reply
traveled the entire reverse path. Eight namespace crossings, three
encapsulations, and six protocol interactions — all in a few milliseconds.

### 6.2 Layer-by-Layer Verification

Run these in order. If any layer fails, the layers above it won't work.
Debug from the bottom up.

**Layer 1 — IS-IS adjacencies:**

```bash
sudo podman exec clab-srv6-lab-spine vtysh -c 'show isis neighbor'
sudo podman exec frr-pe-router vtysh -c 'show isis neighbor'
```

All adjacencies should be `Up`. If any are missing, check that `eth1` is in the
pe-router namespace and IS-IS is enabled on it.

**Layer 2 — SRv6 locator reachability:**

```bash
sudo podman exec clab-srv6-lab-spine vtysh -c 'show ipv6 route isis'
# fd00:30:{16,17,18}::/48 must be present
```

These /48 routes are the SRv6 locators. Without them, SRv6 packets have no
forwarding path.

**Layer 3 — BGP L3VPN sessions:**

```bash
sudo podman exec clab-srv6-lab-spine vtysh -c 'show bgp ipv4 vpn summary'
# All 3 peers established, PfxRcd = 5
```

If sessions aren't established, check IS-IS loopback reachability (BGP peers
over loopbacks).

**Layer 4 — CMN VRF populated:**

```bash
sudo podman exec frr-pe-router vtysh -c 'show ip route vrf CMN'
# Remote subnets with seg6 encapsulation
```

Look for `B>` routes with `seg6` next-hops. These are the VPN-imported routes.

**Layer 5 — VTEP reachability (critical checkpoint):**

```bash
sudo ip netns exec l2-vxlan ping -c 1 10.55.96.154
# Must work before EVPN can establish
```

This single ping tests the entire L3 path: l2-vxlan → veth → CMN VRF → SRv6 →
spine → PE2 → CMN VRF → veth → l2-vxlan. If it fails, EVPN cannot establish TCP
sessions.

**Layer 6 — EVPN sessions:**

```bash
sudo podman exec frr-l2-vxlan vtysh -c 'show bgp l2vpn evpn summary'
# Both peers established
```

**Layer 7 — VXLAN FDB entries:**

```bash
sudo ip netns exec l2-vxlan bridge fdb show dev vxlan0
# Remote MAC entries with dst = remote VTEP IPs
```

BUM entries (all-zeros MAC) should exist for each remote VTEP. Specific MAC
entries appear after traffic flows or after EVPN Type-2 routes are received.

**Layer 8 — Machine network ping:**

```bash
ping -c 3 10.55.99.8
```

### 6.3 Watch Packets Live

Open three terminals on PE1 to see the same packet at three different layers:

**Terminal 1 — ICMP on the host veth (Layer 7 — original packet):**
```bash
sudo ip netns exec l2-vxlan tcpdump -i ve-l2-br-home -nn icmp
```

You'll see the original ICMP echo request/reply with the machine network
addresses (10.55.99.7 → 10.55.99.8).

**Terminal 2 — VXLAN encap entering pe-router (Layer 4 — VXLAN-wrapped):**
```bash
sudo ip netns exec pe-router tcpdump -i ve-pe-cmn-l2 -nn udp port 4789
```

You'll see UDP packets to port 4789 — the VXLAN-encapsulated frames with
VTEP source/destination IPs.

**Terminal 3 — SRv6 on the fabric link (Layer 2 — double-encapsulated):**
```bash
sudo ip netns exec pe-router tcpdump -i eth1 -nn ip6
```

You'll see IPv6 packets with SRv6 source/destination addresses — the VXLAN
packets wrapped in SRv6 for fabric transit.

Then from another session:
```bash
ping -c 1 10.55.99.8
```

You'll see the packet appear in Terminal 1 as ICMP, in Terminal 2 as
VXLAN-encapped, and in Terminal 3 as an SRv6 IPv6 packet — the same data
at three different layers of the stack.

### 6.4 Congratulations

You've built a complete service provider-grade network fabric from scratch:

- **4 nodes**, 3 links
- **2 network namespaces** per PE, 4 veth pairs each
- **IS-IS** for underlay reachability (Level-1, IPv6)
- **SRv6 with micro-SID** for transport encapsulation (End.DT4 for VRF delivery)
- **BGP L3VPN** for VRF route distribution (spine as route reflector, AS 65500)
- **BGP EVPN** for MAC/IP learning (PE1 as route reflector, AS 65501)
- **VXLAN** for L2 overlay with anycast gateway (VNI 330, VLAN 30)
- **Double encapsulation:** Ethernet → VXLAN → SRv6 → fabric

The same architecture runs in production on bare metal clusters. The
only differences are: more PEs, redundant BLs, physical NICs instead of veths
to the host, and automation scripts instead of manual commands.
