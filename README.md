# BADASS — BGP At Doors of Autonomous Systems is Simple

Network administration project: configuring VXLAN tunneling and BGP EVPN routing in a simulated data center using GNS3 with Docker.

## Overview

This project is part of the 42 curriculum and is divided into three progressive parts. Starting from basic Docker/GNS3 integration, it builds up to a full BGP EVPN data center fabric with automatic MAC learning over VXLAN tunnels.

## Parts

### Part 1 — GNS3 + Docker

Two custom Docker images used throughout the project:

- **Host image** — Alpine Linux with busybox and networking tools (iproute2, iputils, tcpdump)
- **Router image** — FRRouting with four active daemons:
  - `zebra` — interface and routing table management
  - `bgpd` — BGP routing
  - `ospfd` — OSPF routing
  - `isisd` — IS-IS routing engine

Both images are imported into GNS3 and connected in a simple topology to verify basic connectivity.

### Part 2 — VXLAN

A VXLAN (RFC 7348) overlay network with VNI 10 that extends a Layer 2 segment across two routers.

**Static mode** — each router has a point-to-point VXLAN tunnel configured with `remote <peer_ip>`. A bridge (`br0`) joins the VXLAN interface with the host-facing port, placing both hosts in the same L2 domain.

**Dynamic multicast mode** — the static `remote` parameter is replaced with `group 239.1.1.1`. Routers discover peers automatically via IGMP, enabling dynamic BUM (Broadcast, Unknown unicast, Multicast) traffic flooding.

```
host_1 ── router_1 ──── router_2 ── host_2
              └── VXLAN VNI 10 ──┘
```

### Part 3 — BGP EVPN

A small data center fabric using BGP EVPN (RFC 7432) with VXLAN as the data plane.

**Architecture:**
- 1 Route Reflector (RR) — central iBGP peer that distributes EVPN routes
- 3 VTEPs (leaf switches) — encapsulate/decapsulate VXLAN traffic
- 3 hosts — connected behind VTEPs

```
              Route Reflector
             /      |       \
        VTEP_2   VTEP_3   VTEP_4
          |                  |
       host_1             host_3
```

**Underlay** — OSPF ensures loopback-to-loopback reachability between all routers.

**Overlay** — iBGP (AS 65000) with EVPN address family. The RR acts as a route-reflector-client for all VTEPs. Each VTEP advertises its VNI (`advertise-all-vni`), and MAC addresses are learned automatically through the control plane:
- **Type 3 routes** (IMET) — pre-configured flood routes, present before any host is connected
- **Type 2 routes** (MAC/IP) — created dynamically when a host comes online, distributed by the RR to all VTEPs

## Repository structure

```
badass/
├── P1/
│   ├── Dockerfile.host          # Alpine + busybox
│   ├── Dockerfile.router        # FRRouting (zebra, bgpd, ospfd, isisd)
│   ├── daemons                  # FRR daemon toggle config
│   ├── _dmiasnik-1_host         # Host config
│   └── _dmiasnik-2              # Router config
├── P2/
│   ├── _dmiasnik-1_host         # Host 1 config
│   ├── _dmiasnik-2_host         # Host 2 config
│   ├── _dmiasnik-1_s            # Router 1 — static VXLAN
│   ├── _dmiasnik-2_s            # Router 2 — static VXLAN
│   ├── _dmiasnik-1_g            # Router 1 — multicast VXLAN
│   └── _dmiasnik-2_g            # Router 2 — multicast VXLAN
└── P3/
    ├── _sshevche-1              # Route Reflector (OSPF + BGP EVPN)
    ├── _sshevche-2              # VTEP 2 (OSPF + BGP + VXLAN)
    ├── _sshevche-3              # VTEP 3
    ├── _sshevche-4              # VTEP 4
    ├── _sshevche-1_host         # Host behind VTEP 2
    ├── _sshevche-2_host         # Host behind VTEP 3
    └── _sshevche-3_host         # Host behind VTEP 4
```

## Key technologies

| Technology | Purpose |
|---|---|
| GNS3 | Network simulation platform |
| Docker | Containerized network nodes |
| FRRouting | Routing daemon suite (fork of Quagga) |
| VXLAN (RFC 7348) | L2 overlay tunneling over L3 underlay |
| BGP EVPN (RFC 7432) | Control plane for VXLAN — MAC/IP route distribution |
| OSPF | Interior gateway protocol for underlay reachability |
| IS-IS | Alternative IGP routing engine |

## Authors

- **dmiasnik** — Part 1, Part 2
- **sshevche** — Part 3
