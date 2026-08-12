# BADASS — Testing Guide

## Prerequisites

- VM running with Docker and GNS3 installed
- Docker images `dmiasnik-host` and `dmiasnik-router` built
- GNS3 projects for P1, P2, P3 created and running

---

## Part 1 — GNS3 + Docker

### Test 1.1: Docker images exist

```bash
docker images | grep dmiasnik-host
docker images | grep dmiasnik-router
```

Both lines must appear.

### Test 1.2: Host image has busybox

```bash
docker run --rm dmiasnik-host busybox --help
```

Must print busybox usage info.

### Test 1.3: Router image — FRR daemons running

```bash
docker run --rm -d --name test-router dmiasnik-router
sleep 3
docker exec test-router ps aux | grep -E "zebra|bgpd|ospfd|isisd"
docker exec test-router vtysh -c "show version"
docker stop test-router
```

All 4 daemons (zebra, bgpd, ospfd, isisd) must be listed.
`vtysh` must respond with FRRouting version.

### Test 1.4: Connectivity in GNS3

Open GNS3, start P1 project. On `_dmiasnik-1_host`:

```bash
ping 10.1.1.2
```

Must receive ICMP replies from `_dmiasnik-2`.

---

## Part 2 — VXLAN

### Test 2.1: Static VXLAN — VXLAN interface exists

On both routers:

```bash
ip -d link show vxlan10
```

Must show: `vxlan id 10`, `dstport 4789`, `remote <peer_ip>`.

### Test 2.2: Static VXLAN — Bridge configured

```bash
brctl show br0
```

Must list `vxlan10` and `eth1` as bridge ports.

### Test 2.3: Static VXLAN — Host-to-host ping

On `_dmiasnik-1_host`:

```bash
ping 30.1.1.2
```

Must receive replies from `_dmiasnik-2_host`.

### Test 2.4: Static VXLAN — Traffic inspection

On either router, capture traffic on eth0:

```bash
tcpdump -i eth0 -n port 4789
```

While pinging between hosts — must see UDP packets on port 4789 with VXLAN headers.

### Test 2.5: Dynamic Multicast — Group membership

On both routers:

```bash
ip maddr show dev eth0
```

Must show group `239.1.1.1`.

### Test 2.6: Dynamic Multicast — MAC table

```bash
bridge fdb show dev vxlan10
```

Must show learned MAC addresses with VXLAN destination.

### Test 2.7: Dynamic Multicast — Host-to-host ping

On `_dmiasnik-1_host`:

```bash
ping 30.1.1.2
```

Must receive replies.

---

## Part 3 — BGP EVPN

### Test 3.1: OSPF neighbors established

On RR (`_sshevche-1`):

```bash
vtysh -c "show ip ospf neighbor"
```

Must show 3 neighbors (1.1.1.2, 1.1.1.3, 1.1.1.4) in state `Full`.

### Test 3.2: Loopback reachability

On any VTEP:

```bash
ping 1.1.1.1
ping 1.1.1.2
ping 1.1.1.3
ping 1.1.1.4
```

All 4 loopbacks must be reachable.

### Test 3.3: BGP sessions up

On RR (`_sshevche-1`):

```bash
vtysh -c "show bgp summary"
```

Must show 3 peers (1.1.1.2, 1.1.1.3, 1.1.1.4) with state `Estab` and non-zero prefixes.

### Test 3.4: EVPN Type 3 routes (before hosts)

On RR, **before starting any host**:

```bash
vtysh -c "show bgp l2vpn evpn route type multicast"
```

Must show Type 3 (IMET) routes from each VTEP. These are pre-configured flood routes for VNI 10.

### Test 3.5: No Type 2 routes (before hosts)

```bash
vtysh -c "show bgp l2vpn evpn route type macip"
```

Must be empty — no MAC/IP routes exist yet.

### Test 3.6: Type 2 route appears (start first host)

Start `_sshevche-1_host` and assign IP. On VTEP `_sshevche-2`:

```bash
vtysh -c "show evpn mac vni 10"
```

Must show the MAC address of `_sshevche-1_host` learned locally.

On RR:

```bash
vtysh -c "show bgp l2vpn evpn route type macip"
```

Must show a new Type 2 route with the MAC of `_sshevche-1_host`.

### Test 3.7: Type 2 route propagated to remote VTEP

On `_sshevche-4` (remote VTEP):

```bash
vtysh -c "show bgp l2vpn evpn route type macip"
```

Must show the Type 2 route for `_sshevche-1_host` MAC, received from RR.

### Test 3.8: Second host — second Type 2 route

Start `_sshevche-3_host`. On RR:

```bash
vtysh -c "show bgp l2vpn evpn route type macip"
```

Must show 2 Type 2 routes now.

### Test 3.9: Ping between hosts through VXLAN + EVPN

On `_sshevche-1_host`:

```bash
ping 20.1.1.3
```

Must receive replies from `_sshevche-3_host`. Traffic traverses:
`host_1 → VTEP_2 → (VXLAN VNI 10) → VTEP_4 → host_3`

### Test 3.10: Traffic inspection

On any VTEP, capture traffic on eth0:

```bash
tcpdump -i eth0 -n
```

While pinging between hosts, must see:
- **VXLAN** packets (UDP port 4789, VNI 10)
- **ICMP** packets inside the VXLAN tunnel
- **OSPF** hello packets (protocol 89)

### Test 3.11: VXLAN details

On any VTEP:

```bash
ip -d link show vxlan10
bridge fdb show dev vxlan10
```

Must show VNI 10, nolearning flag, and forwarding database entries with remote VTEP IPs.

---

## Submission Checklist

```bash
find . -maxdepth 2 -not -path './.git/*' -ls
```

Verify the following files exist:

```
./P1/P1.gns3project        (ZIP archive)
./P1/_dmiasnik-1_host
./P1/_dmiasnik-2

./P2/P2.gns3project        (ZIP archive)
./P2/_dmiasnik-1_g
./P2/_dmiasnik-1_host
./P2/_dmiasnik-1_s
./P2/_dmiasnik-2_g
./P2/_dmiasnik-2_host
./P2/_dmiasnik-2_s

./P3/P3.gns3project        (ZIP archive)
./P3/_sshevche-1
./P3/_sshevche-1_host
./P3/_sshevche-2
./P3/_sshevche-2_host
./P3/_sshevche-3
./P3/_sshevche-3_host
./P3/_sshevche-4
```

Verify `.gns3project` files are ZIP archives:

```bash
file P1/P1.gns3project P2/P2.gns3project P3/P3.gns3project
```

All three must show `Zip archive data`.
