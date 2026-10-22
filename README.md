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

```
+---------------------+                     +---------------------+
|  _dmiasnik-1_host   |                     |     _dmiasnik-2     |
| eth0 10.1.1.1/24    +---- 10.1.1.0/24 ----+ eth0 10.1.1.2/24    |
| Alpine + busybox    |                     | FRRouting:          |
|                     |                     |  zebra, bgpd,       |
|                     |                     |  ospfd, isisd       |
+---------------------+                     +---------------------+
```

| Element | Where | Purpose |
|---|---|---|
| `eth0` | host and router | one shared `/24` network, both ends of the link can reach each other |
| busybox | host | minimal Linux userland with basic networking tools |
| FRRouting | router | routing suite, daemons enabled in `daemons` file |
| `zebra` | router | manages interfaces and the kernel routing table |
| `bgpd`, `ospfd`, `isisd` | router | routing protocol daemons used in later parts |

### Part 2 — VXLAN

A VXLAN (RFC 7348) overlay network with VNI 10 that extends a Layer 2 segment across two routers.

**Static mode** — each router has a point-to-point VXLAN tunnel configured with `remote <peer_ip>`. A bridge (`br0`) joins the VXLAN interface with the host-facing port, placing both hosts in the same L2 domain.

**Dynamic multicast mode** — the static `remote` parameter is replaced with `group 239.1.1.1`. Routers discover peers automatically via IGMP, enabling dynamic BUM (Broadcast, Unknown unicast, Multicast) traffic flooding.

```
+---------------------+                     +---------------------+
|     _dmiasnik-1     |                     |     _dmiasnik-2     |
| eth0 10.1.1.1/24    +---- 10.1.1.0/24 ----+ eth0 10.1.1.2/24    |
| vxlan10  VNI 10     +=== VXLAN VNI 10 ====+ vxlan10  VNI 10     |
| _s: remote 10.1.1.2 |                     | _s: remote 10.1.1.1 |
| _g: group 239.1.1.1 |                     | _g: group 239.1.1.1 |
| br0: eth1 + vxlan10 |                     | br0: eth1 + vxlan10 |
| eth1  no IP         |                     | eth1  no IP         |
+----------+----------+                     +----------+----------+
           |                                           |
+----------+----------+                     +----------+----------+
|  _dmiasnik-1_host   |                     |  _dmiasnik-2_host   |
| eth0 30.1.1.1/24    |                     | eth0 30.1.1.2/24    |
+---------------------+                     +---------------------+
```

| Element | Where | Purpose |
|---|---|---|
| `eth0` with `10.1.1.0/24` | routers | underlay link, carries encapsulated VXLAN packets |
| `vxlan10` | routers | tunnel endpoint, wraps L2 frames into UDP port 4789 with VNI 10 |
| `remote` | routers, `_s` files | static mode: the peer address is hardcoded |
| `group 239.1.1.1` | routers, `_g` files | multicast mode: peers join the group and receive BUM traffic |
| `br0` | routers | bridge joining the host port with the tunnel |
| `eth1` without IP | routers | host-facing port, acts as a switch port |
| `eth0` with `30.1.1.0/24` | hosts | both hosts share one L2 segment through the tunnel |

**Inside a router.** The bridge `br0` has two ports: `eth1` (towards the host) and `vxlan10` (towards the tunnel). `eth0` is not a bridge port: it is a regular IP interface underneath the tunnel, and `vxlan10` sends its UDP packets through it (`dev eth0`).

```
             bridge br0
         +---------------+
host ----+ eth1  vxlan10 +
         +-----------+---+
                     | encapsulation into UDP 4789
                   eth0 (10.1.1.1) ------- peer router
```

| Port | Bridge member | Role |
|---|---|---|
| `eth1` | yes | receives and delivers the host's Ethernet frames |
| `vxlan10` | yes | encapsulates frames into UDP and decapsulates incoming ones |
| `eth0` | no | underlay IP interface, physically carries the UDP packets |

Path of a frame from the host:

1. The frame arrives on `eth1`.
2. `br0` forwards it to its other port, `vxlan10`.
3. `vxlan10` wraps it into a UDP packet addressed to the peer (`10.1.1.2`).
4. The packet leaves through `eth0` as ordinary IP traffic.

The peer router does the reverse: `eth0` → `vxlan10` decapsulates → `br0` → `eth1` → host.

`eth0` must stay out of the bridge: an interface enslaved to a bridge stops serving its own IP address, so `vxlan10` could no longer send or receive UDP and the tunnel would break. `eth0` and `eth1` are created by GNS3 from the node's adapter count; the script only creates `vxlan10` (`ip link add`) and `br0` (`brctl addbr`).

### Part 3 — BGP EVPN

A small data center fabric using BGP EVPN (RFC 7432) with VXLAN as the data plane.

**Architecture:**
- 1 Route Reflector (RR) — central iBGP peer that distributes EVPN routes
- 3 VTEPs (leaf switches) — encapsulate/decapsulate VXLAN traffic
- 3 hosts — connected behind VTEPs

```
                          +---------------------+
                          |     _sshevche-1     |
                          | Route Reflector     |
                          | lo   1.1.1.1/32     |
                          | eth0 10.1.1.1/30    |
                          | eth1 10.1.1.5/30    |
                          | eth2 10.1.1.9/30    |
                          | OSPF area 0         |
                          | BGP AS 65000, RR    |
                          | no VXLAN, no br0    |
                          +----------+----------+
           +-------------------------+-------------------------+
           | RR eth0                 | RR eth1                 | RR eth2
           | 10.1.1.0/30             | 10.1.1.4/30             | 10.1.1.8/30
+----------+----------+   +----------+----------+   +----------+----------+
|     _sshevche-2     |   |     _sshevche-3     |   |     _sshevche-4     |
| leaf / VTEP         |   | leaf / VTEP         |   | leaf / VTEP         |
| lo   1.1.1.2/32     |   | lo   1.1.1.3/32     |   | lo   1.1.1.4/32     |
| eth0 10.1.1.2/30    |   | eth0 10.1.1.6/30    |   | eth0 10.1.1.10/30   |
| OSPF area 0         |   | OSPF area 0         |   | OSPF area 0         |
| BGP AS 65000 -> RR  |   | BGP AS 65000 -> RR  |   | BGP AS 65000 -> RR  |
| vxlan10  VNI 10     |   | vxlan10  VNI 10     |   | vxlan10  VNI 10     |
| br0: eth1 + vxlan10 |   | br0: eth1 + vxlan10 |   | br0: eth1 + vxlan10 |
| eth1  no IP         |   | eth1  no IP         |   | eth1  no IP         |
+----------+----------+   +----------+----------+   +----------+----------+
           |                         |                         |
+----------+----------+   +----------+----------+   +----------+----------+
|  _sshevche-1_host   |   |  _sshevche-2_host   |   |  _sshevche-3_host   |
| eth0 20.1.1.1/24    |   | eth0 20.1.1.2/24    |   | eth0 20.1.1.3/24    |
| no FRR, no VXLAN    |   | no FRR, no VXLAN    |   | no FRR, no VXLAN    |
+---------------------+   +---------------------+   +---------------------+
```

| Element | Where | Purpose |
|---|---|---|
| `lo` | all routers | stable router address, BGP sessions and VXLAN tunnels run between loopbacks |
| `eth0` with `/30` | all routers | leaf to RR link, a separate small network per link |
| OSPF | all routers | makes every loopback reachable from every router |
| BGP | all routers | leaves exchange VNI membership and MAC locations through the RR |
| Route Reflector | `_sshevche-1` only | reflects BGP updates from one leaf to the others |
| `vxlan10` | leaves only | encapsulates and decapsulates host frames in UDP |
| `br0` | leaves only | bridge joining the host port with the tunnel |
| `eth1` without IP | leaves only | host-facing port, acts as a switch port |
| `eth0` with `/24` | hosts only | all hosts share one network `20.1.1.0/24` |

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
| VXLAN ([RFC 7348](https://datatracker.ietf.org/doc/html/rfc7348)) | L2 overlay tunneling over L3 underlay |
| BGP ([RFC 4271](https://datatracker.ietf.org/doc/html/rfc4271)) | Base routing protocol |
| MP-BGP ([RFC 4760](https://datatracker.ietf.org/doc/html/rfc4760)) | Multiprotocol extensions (EVPN address family) |
| BGP EVPN ([RFC 7432](https://datatracker.ietf.org/doc/html/rfc7432)) | Control plane for VXLAN — MAC/IP route distribution |
| OSPF ([RFC 2328](https://datatracker.ietf.org/doc/html/rfc2328)) | Interior gateway protocol for underlay reachability |
| IS-IS ([RFC 1195](https://datatracker.ietf.org/doc/html/rfc1195)) | Alternative IGP routing engine |

Full RFC list with links: [`docs/RFC.md`](docs/RFC.md)
