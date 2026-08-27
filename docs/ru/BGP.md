# Что такое BGP

## Простыми словами

**BGP** расшифровывается как **Border Gateway Protocol** — «протокол граничного шлюза». Описан в [RFC 4271](https://datatracker.ietf.org/doc/html/rfc4271).

Это протокол, которым большие сети договариваются: «какие сети через меня достижимы». На BGP держится маршрутизация **между** провайдерами в Интернете. В дата-центрах тот же BGP часто используют **внутри** фабрики — в том числе чтобы разносить не только IP-префиксы, а ещё MAC-адреса (это уже EVPN через MP-BGP).

Аналогия: OSPF — карта дорог *внутри одного города*. BGP — указатели на границах *между городами и странами*: «до этой сети лучше ехать через соседа A, а не через B».

У нас в проекте:

- Part 1 — демон **bgpd** должен быть запущен;
- Part 3 — BGP реально настраиваем: AS 65000, iBGP к Route Reflector, семейство `l2vpn evpn`.

---

## Зачем BGP нужен в мире

Интернет — не одна сеть, а тысячи **автономных систем (AS)**: провайдеры, облака, университеты, корпорации.

У каждой AS свой номер (**ASN**). Например, у Google — 15169. Между AS нужен общий язык «какие префиксы я объявляю и на каких условиях». Этим языком стал BGP.

Без BGP не было бы согласованной картины «как пакет из Лозанны доезжает до сервера в другом AS».

---

## AS — автономная система

**AS (Autonomous System)** — группа сетей под одним административным управлением и одной политикой маршрутизации.

**ASN** — номер этой системы. Бывают публичные (в Интернете) и приватные (64512–65534 для внутренней лаборатории).

У нас в проекте все роутеры в **AS 65000** — приватный номер. Между ними поэтому **iBGP**, а не eBGP.

---

## iBGP и eBGP

| | **eBGP** (external) | **iBGP** (internal) |
|--|---------------------|---------------------|
| Между кем | разные AS | одна и та же AS |
| Где в жизни | стык провайдеров | внутри дата-центра / AS |
| В нашем проекте | нет | да, все `remote-as 65000` |

**eBGP** — классика Интернета: AS 100 говорит с AS 200.

**iBGP** — BGP внутри одной AS. Зачем, если уже есть OSPF?  
OSPF умеет IP-маршруты underlay. Нам в Part 3 нужно возить **EVPN** (MAC, VNI). Это делает BGP (через MP-BGP), а не OSPF.

### Проблема iBGP: full mesh

Правило iBGP (упрощённо): маршрут, полученный от одного iBGP-соседа, **нельзя** просто так отдать другому iBGP-соседу (защита от петель).

Следствие: чтобы все знали все маршруты, каждый должен пириться с каждым → **full mesh**. На 4 узлах ещё терпимо, на 100 — кошмар.

Решение в нашем проекте — **Route Reflector (RR)**: все VTEP пирятся только с `_sshevche-1`, а RR отражает маршруты клиентам (`route-reflector-client`). См. [RFC 4456](https://datatracker.ietf.org/doc/html/rfc4456).

---

## Как BGP работает на проводе

BGP бежит поверх **TCP, порт 179** (не «сам по себе по UDP», как OSPF Hello).

Два роутера:

1. Устанавливают TCP-сессию (часто с loopback + `update-source lo`).
2. Меняются сообщением **OPEN** («я AS X, я умею такие capability»).
3. Потом **KEEPALIVE** — «сессия жива».
4. **UPDATE** — «вот новые маршруты» или «эти снимаю».
5. При аварии — **NOTIFICATION** и разрыв.

Пока в `show bgp summary` состояние **Established** — сессия поднята. Пока Idle/Active/Connect — underlay, AS, neighbor IP или firewall/TCP.

У нас underlay до loopback даёт **OSPF**. Сначала Full у OSPF, потом Established у BGP.

---

## Что BGP объявляет: NLRI и атрибуты

В UPDATE два слоя смысла:

- **NLRI** (Network Layer Reachability Information) — *что* достижимо  
  (в классике: префикс `203.0.113.0/24`; в EVPN: Type 2 MAC, Type 3 IMET…).
- **Path attributes** — *как к этому относиться*: next-hop, AS-path, local preference, MED, communities…

BGP — протокол **политик**: можно предпочесть одного соседа другому не потому что путь короче по SPF, а потому что так решила бизнес-логика. В нашей маленькой лабе политики почти не трогаем; важен next-hop VTEP и сам факт EVPN NLRI.

Подробнее: [`docs/NLRI.md`](NLRI.md).

---

## BGP vs OSPF (IGP)

| | OSPF (`ospfd`) | BGP (`bgpd`) |
|--|----------------|--------------|
| Класс | IGP | EGP по происхождению (между AS); внутри AS — iBGP |
| Транспорт | IP proto 89, Hello | TCP 179 |
| Типичный груз | IP-линки, loopback | IP-префиксы **или** EVPN |
| Сходимость | обычно быстрее внутри площадки | заточен под масштаб и политику |
| В проекте Part 3 | underlay | EVPN overlay control plane |

---

## MP-BGP — расширение, без которого нет EVPN

Классический BGP (RFC 4271) по сути про IPv4 unicast.

**MP-BGP** ([RFC 4760](https://datatracker.ietf.org/doc/html/rfc4760)) добавляет **address family**: по одной TCP-сессии можно возить разные типы NLRI.

В Part 3:

```text
address-family l2vpn evpn
 neighbor … activate
```

Мы даже отключаем дефолтный IPv4 unicast:

```text
no bgp default ipv4-unicast
```

чтобы BGP не путал с underlay — underlay уже на OSPF.

Подробнее: [`docs/MP-BGP.md`](MP-BGP.md), [`docs/BGP_EVPN.md`](BGP_EVPN.md) или [`docs/ru/BGP_EVPN.md`](ru/BGP_EVPN.md).

---

## BGP в конфигурации (Part 3)

### Общее

```text
router bgp 65000
 bgp router-id 1.1.1.X
 no bgp default ipv4-unicast
 neighbor <lo> remote-as 65000
 neighbor <lo> update-source lo
```

- `65000` — наша AS → iBGP;
- `router-id` — уникальный ID (обычно loopback);
- `remote-as 65000` — сосед в той же AS;
- `update-source lo` — TCP с адреса loopback (стабильный peering).

### Route Reflector (`_sshevche-1`)

Соседи — три VTEP. В `l2vpn evpn`: `activate` + `route-reflector-client`.  
RR сам VXLAN может не иметь: он почтальон маршрутов.

### VTEP (`_sshevche-2/3/4`)

Один сосед — RR. В `l2vpn evpn`: `activate` + `advertise-all-vni`  
(объявить локальные VNI → появятся Type 3, потом Type 2 с MAC хостов).

---

## Полезные команды

```bash
vtysh -c "show bgp summary"
vtysh -c "show bgp neighbors"
vtysh -c "show bgp l2vpn evpn"
vtysh -c "show bgp l2vpn evpn route type multicast"   # Type 3
vtysh -c "show bgp l2vpn evpn route type macip"       # Type 2
```

Процесс демона:

```bash
ps aux | grep bgpd
```

Демон ≠ протокол: демон — [`docs/BGPD.md`](BGPD.md).

---

## RFC

| RFC | Тема | Ссылка |
|-----|------|--------|
| **4271** | BGP-4 | https://datatracker.ietf.org/doc/html/rfc4271 |
| **4760** | MP-BGP | https://datatracker.ietf.org/doc/html/rfc4760 |
| **4456** | Route Reflection | https://datatracker.ietf.org/doc/html/rfc4456 |
| **7432** | EVPN | https://datatracker.ietf.org/doc/html/rfc7432 |

Сводная таблица: [`docs/ru/RFC.md`](ru/RFC.md) (и при наличии корневой `docs/RFC.md`).

---

## Кратко

BGP — протокол обмена маршрутной информацией между (и внутри) автономными системами, поверх TCP 179. В Интернете это склейка провайдеров; у нас в Part 3 — iBGP AS 65000 с Route Reflector. Через MP-BGP семейство `l2vpn evpn` BGP разносит Type 3 (VNI) и Type 2 (MAC). Достижимость loopback для сессий даёт OSPF. Демон, который всё это крутит — `bgpd`.
