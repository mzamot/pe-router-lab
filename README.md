# SRv6 + EVPN VXLAN Lab

Networking lab to explore SRv6 + BGP L3VPN + BGP EVPN/VXLAN architecture using containerlab and Ansible.

```
            ┌───────────────┐
            │    spine      │   FRR container (iBGP route reflector)
            │  fc00::1      │
            └─┬────┬────┬───┘
             ╱     │     ╲        IS-IS level-1 p2p
            ╱      │      ╲
    ┌───────┐  ┌───────┐  ┌───────┐
    │  pe1  │  │  pe2  │  │  pe3  │   AlmaLinux 10 VMs
    │       │  │       │  │       │
    │  ┌─────────────────────┐    │   Inside each VM:
    │  │ default namespace   │    │    ve-home-l2-br (10.55.99.{7,8,9}/26)
    │  ├─────────────────────┤    │
    │  │ netns: pe-router    │    │    IS-IS + BGP L3VPN + SRv6, VRF CMN/EDN
    │  ├─────────────────────┤    │
    │  │ netns: l2-vxlan     │    │    VXLAN bridge (VNI 330), BGP EVPN
    │  └─────────────────────┘    │    Anycast gw 10.55.99.1
    └───────┘  └───────┘  └───────┘
```

## Prerequisites

- containerlab >= 0.55.0
- vrnetlab AlmaLinux image (`localhost/vrnetlab/rhel_almalinux:10`)
- ansible-core
- sshpass

## Usage

### Deploy

```bash
sudo ansible-playbook deploy.yml
```

This runs four plays in sequence:

1. **Deploy containerlab topology** — creates the spine container and 3 PE VMs
2. **PE prerequisites** — waits for VMs to boot, installs packages, loads kernel modules
3. **Configure PEs** — templates FRR configs, builds namespaces/veths/VXLAN, starts FRR containers
4. **Verify** — checks IS-IS adjacencies, BGP VPN sessions, and end-to-end EVPN connectivity

### Re-configure PEs without re-deploying containerlab

```bash
sudo ansible-playbook deploy.yml --tags configure
```

### Run verification only

```bash
sudo ansible-playbook deploy.yml --tags verify
```

### Teardown

```bash
sudo ansible-playbook teardown.yml
```

## Inspecting the lab

SSH into any PE (password: `clab@123`):

```bash
ssh clab@clab-srv6-lab-pe1
```

PE router FRR (IS-IS, BGP L3VPN, SRv6):

```bash
sudo podman exec frr-pe-router vtysh
  show isis neighbor
  show bgp ipv4 vpn
  show ip route vrf CMN
  show segment-routing srv6 locator
```

L2 VXLAN FRR (BGP EVPN):

```bash
sudo podman exec frr-l2-vxlan vtysh
  show bgp l2vpn evpn summary
  show evpn vni
```

Spine route reflector (from the lab host):

```bash
sudo podman exec clab-srv6-lab-spine vtysh
  show isis neighbor
  show bgp ipv4 vpn summary
```

## Bare-metal deployment

A separate inventory and playbook exist for deploying to physical or VM hosts where the spine is also managed by Ansible:

```bash
ansible-playbook -i baremetal/inventory.yml baremetal/deploy.yml
ansible-playbook -i baremetal/inventory.yml baremetal/teardown.yml
```

Both paths share the same Jinja templates in `templates/`.

## File structure

```
├── deploy.yml               Ansible playbook (containerlab)
├── teardown.yml              Destroy the lab
├── inventory.yml             Per-node addressing and credentials
├── ansible.cfg               SSH settings for containerlab VMs
├── topology.clab.yml         Containerlab topology definition
├── templates/                Shared Jinja templates
│   ├── pe-router-frr.conf.j2
│   ├── pe-router-daemons.j2
│   ├── l2-vxlan-frr.conf.j2
│   ├── l2-vxlan-daemons.j2
│   ├── spine-frr.conf.j2        (baremetal only)
│   ├── spine-daemons.j2         (baremetal only)
│   └── setup-pe-networking.sh.j2
├── configs/spine/            Static spine config (bind-mounted by containerlab)
├── baremetal/                Bare-metal inventory and playbooks
└── learning/                 Educational deep-dives
```

## Learning resources

- [Foundations](learning/01-foundations.md) — Linux networking primitives, namespaces, VRFs, veths
- [IS-IS](learning/02-isis.md) — The underlay routing protocol
- [SRv6](learning/03-srv6.md) — Segment Routing over IPv6
- [BGP and L3VPN](learning/04-bgp-l3vpn.md) — BGP VPN families, route distinguishers, route targets
- [VXLAN and BGP EVPN](learning/05-vxlan-evpn.md) — L2 overlay, VNI, anycast gateway
- [Full Packet Walk](learning/06-packet-walk.md) — End-to-end packet path through the architecture
- [How All Protocols Work Together](learning/07-protocols-together.md) — Tying it all together

For a manual step-by-step build guide, see [HOWTO.md](HOWTO.md).
