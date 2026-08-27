# Что такое MP-BGP

## Простыми словами

**MP-BGP** расшифровывается как **Multi-Protocol BGP** — «многопротокольный BGP». Описан в RFC 4760.

Обычный BGP (RFC 4271) умел возить по сути одно: «я знаю IPv4-сеть вот с таким префиксом». Как словарь только на одном языке.

MP-BGP — это тот же BGP (те же TCP-сессии, те же соседи, тот же AS), но словарь расширили. По одной сессии можно возить IPv4, IPv6, VPN-маршруты и **EVPN** (информацию про Ethernet/MAC).

В проекте мы почти не гоняем IPv4 через BGP. IPv4-связность даёт OSPF. BGP у нас нужен как транспорт для EVPN. Без MP-BGP так сделать нельзя: классический BGP не понимает «MAC за VTEP».

---

## Зачем понадобилось расширение

Интернет сначала был про IPv4 unicast. BGP родился под это.

Потом появились задачи, которых в RFC 4271 нет:

- маршруты IPv6;
- MPLS L3 VPN (разные клиенты, разные таблицы);
- multicast;
- EVPN — объявления MAC и VNI.

Делать отдельный протокол «BGP для MAC» никто не хотел: peering, политики, RR, атрибуты уже есть. Проще сказать: BGP остаётся, а *что лежит внутри UPDATE* — зависит от семейства адресов.

Так и появился MP-BGP.

---

## Address family — «канал» внутри сессии

Ключевое слово MP-BGP — **AFI / SAFI**, на практике в конфиге — **address-family**.

Представь одну TCP-сессию на порт 179. Внутри неё несколько полок:

| Address family | Что везёт | Нужно ли в BADASS |
|----------------|-----------|-------------------|
| `ipv4 unicast` | обычные IPv4-префиксы | нет (это делает OSPF) |
| `ipv6 unicast` | IPv6-префиксы | нет |
| `vpnv4` / L3VPN | IPv4 + метка VRF | нет |
| `l2vpn evpn` | EVPN NLRI (MAC, IMET, …) | **да** |

Сессия может быть Established, а EVPN при этом молчать: сосед есть, а семейство не активировали.

Поэтому в конфиге две ступени:

1. `neighbor 1.1.1.1 remote-as 65000` — вообще пиримся.
2. внутри `address-family l2vpn evpn` → `neighbor ... activate` — по этой сессии включать именно EVPN.

---

## Почему у нас `no bgp default ipv4-unicast`

Во FRR (и во многих Cisco) новый сосед по умолчанию попадает в IPv4 unicast.

Нам IPv4 по BGP не нужен: петли и линки уже в OSPF. Если оставить дефолт, в `show bgp summary` появятся IPv4-префиксы или «нулевые» IPv4 capability, и легко запутаться.

```
no bgp default ipv4-unicast
```

говорит: ничего не активируй сам. Дальше явно:

```
address-family l2vpn evpn
 neighbor 1.1.1.2 activate
```

Это чистый MP-BGP: сессия одна, полезный груз — только EVPN.

---

## Как это выглядит на проводе (идея, без дампа)

Сообщения BGP те же: OPEN, KEEPALIVE, UPDATE, NOTIFICATION.

В OPEN соседи договариваются capability: «я умею MP-BGP, вот мои AFI/SAFI». Если оба умеют `l2vpn evpn`, можно слать EVPN UPDATE.

В UPDATE для MP-BGP появляются атрибуты вроде MP_REACH_NLRI / MP_UNREACH_NLRI: «достижимо / снято с объявления» плюс **NLRI** выбранного семейства.

---

## MP-BGP и iBGP / Route Reflector

MP-BGP не отменяет правила iBGP.

У нас все в AS 65000 — это iBGP. Маршрут от одного iBGP-соседа по умолчанию нельзя отдать другому (антипетля). Поэтому RR:

```
neighbor 1.1.1.2 route-reflector-client
```

RR отражает **EVPN-маршруты** так же, как отражал бы IPv4. Меняется только содержимое. Full mesh VTEP не нужен.

`update-source lo` тоже не «фича EVPN». Это обычный BGP: пакеты сессии с loopback, а OSPF знает, как до loopback доехать. MP-BGP едет внутри этой сессии.

---

## MP-BGP — не то же самое, что EVPN

Путают часто, потому что в лабе они всегда вместе.

- **MP-BGP** — *как* BGP возит разные типы маршрутов (рамка).
- **EVPN** — *какой* тип маршрутов мы возим (картина в рамке): Type 2, Type 3, MAC, VNI.

Можно включить MP-BGP и возить только IPv6 — EVPN при этом нет. Можно говорить про EVPN, но без MP-BGP его в BGP не запихнуть.

В проекте фраза «BGP EVPN» = MP-BGP + семейство l2vpn evpn + VXLAN dataplane.

---

## Где это в наших конфигах

Route Reflector `_sshevche-1`: три соседа, у каждого `activate` + `route-reflector-client` в `address-family l2vpn evpn`.

VTEP `_sshevche-2` (и 3, 4): один сосед — RR, `activate` + `advertise-all-vni`.

`advertise-all-vni` уже про EVPN (что объявлять), не про MP-BGP как таковой. MP-BGP здесь — сам блок `address-family`.

Проверка, что семейство живое:

```bash
vtysh -c "show bgp summary"
vtysh -c "show bgp l2vpn evpn"
```

Если summary зелёный, а `show bgp l2vpn evpn` пустой — часто забыли activate / advertise-all-vni, то есть MP-канал не используется или VTEP молчит.

---

## Коротко, что сказать на защите

Классический BGP возит только IPv4. MP-BGP (RFC 4760) добавляет address family. Мы открываем семейство l2vpn evpn и по тем же iBGP-сессиям с RR передаём EVPN. IPv4 unicast в BGP отключаем, потому что underlay делает OSPF.
