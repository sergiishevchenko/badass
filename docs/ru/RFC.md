# RFC — ссылки по проекту

Ниже — все RFC, которые упоминаются в наших доках и в задании, плюс несколько смежных, полезных на защите.  
Официальные тексты: [datatracker.ietf.org](https://datatracker.ietf.org/) / [rfc-editor.org](https://www.rfc-editor.org/).

---

## Обязательные

| RFC | Тема | Зачем в BADASS | Ссылка |
|-----|------|----------------|--------|
| **RFC 4271** | BGP-4 | Базовый BGP; iBGP/eBGP, UPDATE, peering | https://datatracker.ietf.org/doc/html/rfc4271 |
| **RFC 4760** | MP-BGP (Multiprotocol Extensions for BGP-4) | Address family; без этого нет `l2vpn evpn` | https://datatracker.ietf.org/doc/html/rfc4760 |
| **RFC 7348** | VXLAN | Туннель L2 over UDP 4789, VNI, VTEP (Part 2 и dataplane Part 3) | https://datatracker.ietf.org/doc/html/rfc7348 |
| **RFC 7432** | BGP EVPN | Type 2 / Type 3, MAC в control plane (Part 3) | https://datatracker.ietf.org/doc/html/rfc7432 |
| **RFC 2328** | OSPFv2 | Underlay, area 0, соседи Full (Part 3) | https://datatracker.ietf.org/doc/html/rfc2328 |

В introduction задания прямо названы: BGP (**RFC 4271**), MP-BGP (**RFC 4760**), дальше по тексту VXLAN и BGP EVPN (**RFC 7348**, **RFC 7432**). OSPF в задании для оценки — стандартно опирается на **RFC 2328**.

---

## Сильно связанные с нашей конфигурацией

| RFC | Тема | Зачем | Ссылка |
|-----|------|-------|--------|
| **RFC 4456** | BGP Route Reflection | Зачем RR вместо iBGP full mesh | https://datatracker.ietf.org/doc/html/rfc4456 |
| **RFC 4271** §… / практика iBGP | (см. 4271) | Правило «не ретранслировать iBGP→iBGP» без RR/confederation | https://datatracker.ietf.org/doc/html/rfc4271 |
| **RFC 1195** | Use of OSI IS-IS for Routing in TCP/IP | Integrated IS-IS (IP поверх IS-IS); контекст `isisd` | https://datatracker.ietf.org/doc/html/rfc1195 |
| **ISO/IEC 10589** | IS-IS протокол (не IETF RFC) | «Родной» стандарт IS-IS | часто ищут как ISO 10589; обзор: https://datatracker.ietf.org/doc/html/rfc1195 |

---

## Смежные (ARP, multicast, транспорт)

| RFC | Тема | Зачем в лабе | Ссылка |
|-----|------|--------------|--------|
| **RFC 826** | ARP | Зачем BUM/broadcast до первого ping | https://datatracker.ietf.org/doc/html/rfc826 |
| **RFC 768** | UDP | Outer-транспорт VXLAN | https://datatracker.ietf.org/doc/html/rfc768 |
| **RFC 793** | TCP | BGP сидит на TCP (порт 179) | https://datatracker.ietf.org/doc/html/rfc793 |
| **RFC 3376** | IGMPv3 | Подписка на multicast-группу VXLAN (`group 239.1.1.1`) | https://datatracker.ietf.org/doc/html/rfc3376 |
| **RFC 2236** | IGMPv2 | Более старая версия IGMP, та же идея | https://datatracker.ietf.org/doc/html/rfc2236 |
| **RFC 791** | IPv4 | Outer/inner IP | https://datatracker.ietf.org/doc/html/rfc791 |
| **RFC 894** | IP over Ethernet | Inner Ethernet + IP на хостах | https://datatracker.ietf.org/doc/html/rfc894 |

---

## По темам наших документов

### BGP / BGPD / NLRI / MP-BGP / EVPN
- https://datatracker.ietf.org/doc/html/rfc4271 — BGP  
- https://datatracker.ietf.org/doc/html/rfc4760 — MP-BGP  
- https://datatracker.ietf.org/doc/html/rfc7432 — EVPN  
- https://datatracker.ietf.org/doc/html/rfc4456 — Route Reflector  

См. также: `docs/BGPD.md`, `docs/MP-BGP.md`, `docs/NLRI.md`, `docs/BGP_EVPN.md`

### VXLAN
- https://datatracker.ietf.org/doc/html/rfc7348  

См. также: `docs/VXLAN.md`, `docs/P2_guide.md`

### OSPF
- https://datatracker.ietf.org/doc/html/rfc2328  

См. также: `docs/OSPFD_ISIS.md`, `docs/P3_guide.md`

### IS-IS
- https://datatracker.ietf.org/doc/html/rfc1195  
- ISO 10589 (вне серии IETF RFC)

См. также: `docs/OSPFD_ISIS.md`, `docs/ZEBRA_QUAGGA.md`

### MAC / ARP
- https://datatracker.ietf.org/doc/html/rfc826  

См. также: `docs/MAC.md`

---

## Минимум, который стоит помнить наизусть

1. **4271** — BGP  
2. **4760** — MP-BGP  
3. **7348** — VXLAN  
4. **7432** — EVPN  
5. **2328** — OSPF  
6. **4456** — Route Reflector  
7. **3031** — MPLS
