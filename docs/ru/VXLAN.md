# Что такое VXLAN

## Простыми словами

**VXLAN** расшифровывается как **Virtual Extensible LAN** — «виртуальная расширяемая локальная сеть». Описан в RFC 7348.

Это способ протянуть обычный Ethernet (Layer 2) сквозь IP-сеть (Layer 3). Хосты думают, что сидят на одном свитче: одни MAC, один broadcast, работает ARP. На самом деле между ними роутеры и UDP-туннель.

Картинка: письмо (Ethernet-фрейм хоста) кладут в конверт (UDP-пакет на порт 4789) и отправляют по почте (IP underlay). На том конце конверт снимают и отдают «письмо» как будто оно пришло по кабелю.

В проекте VXLAN — основа Part 2 и data plane Part 3. Part 2 учит сам туннель (static и multicast). Part 3 тот же VNI 10, но «кому слать» решает уже BGP EVPN.

---

## Какую проблему он решает

VLAN (802.1Q) даёт изолированные L2-сегменты, но:

- всего **4096** VLAN ID (12 бит) — мало для облака с тысячами клиентов;
- VLAN плохо «растягивается» через маршрутизируемую сеть: нужен один большой L2-домен, а он хрупкий (штормы, spanning tree).

Дата-центру нужно другое: серверы в разных стойках и даже зданиях, а приложение хочет «одну локалку». Между стойками уже есть нормальная IP-сеть (OSPF, BGP). VXLAN кладёт Ethernet внутрь этой IP-сети и не требует, чтобы весь DC был одним свитчом.

---

## Инкапсуляция — матрёшка пакета

Хост шлёт обычный Ethernet. VTEP не ломает этот фрейм, а **оборачивает**:

```text
[ Outer Ethernet ]  [ Outer IP ]  [ UDP dst 4789 ]  [ VXLAN header + VNI ]
        [ Inner Ethernet ]  [ Inner IP ]  [ ICMP / данные хоста ]
```

**Outer** — то, что видит сеть роутеров. Для OSPF это просто UDP с одного loopback/IP на другой.

**Inner** — то, что видят хосты. Их MAC и IP (30.1.1.x в P2, 20.1.1.x в P3) живут только здесь.

**UDP порт 4789** — номер IANA для VXLAN. В tcpdump на underlay ищешь `port 4789`.

**VXLAN header** несёт **VNI** — какой виртуальный сегмент это был. Без VNI два клиента в одном дата-центре смешались бы в одну кучу.

---

## VNI — номер виртуального свитча

**VNI** = **VXLAN Network Identifier**. 24 бита → больше **16 миллионов** сегментов (2²⁴). VLAN — только 4096.

Одинаковый VNI = один L2-домен. Разный VNI = разные «провода», хосты друг друга не видят.

В проекте везде **VNI 10**, интерфейс обычно зовут `vxlan10`. Имя интерфейса любое, число в `id 10` — обязательное по subject.

---

## VTEP — кто заворачивает и разворачивает

**VTEP** = **VXLAN Tunnel End Point**. У нас это роутер с интерфейсом `vxlan10`.

Он:

1. принимает фрейм от хоста (через bridge `br0` и eth к хосту);
2. пишет VXLAN + UDP + outer IP;
3. отправляет в underlay;
4. на приёме делает обратное и отдаёт фрейм своему хосту.

Хост VTEP-ом не является: он обычный Alpine с одним eth0.

---

## Underlay и overlay

Два этажа, их нельзя путать в IP-адресах.

**Underlay** — IP между роутерами.

- P2: `10.1.1.1` ↔ `10.1.1.2` на eth0.
- P3: линки `/30` плюс loopback `1.1.1.x`, их знает OSPF.

Underlay не знает MAC хостов. Он возит outer-пакеты.

**Overlay** — виртуальный Ethernet VNI 10.

- P2 хосты: `30.1.1.1`, `30.1.1.2`.
- P3 хосты: `20.1.1.1` … `20.1.1.3`.

Overlay не знает, что между роутерами OSPF. Для хоста сосед — просто другой MAC в той же `/24`.

Если ping 30.1.1.2 не идёт, сначала проверяй underlay (`ping 10.1.1.2` с роутера), потом уже vxlan/bridge.

---

## Зачем bridge `br0`

Linux `vxlan10` — это туннель, не «розетка для ПК». Хост висит на `eth1`. Их надо склеить в один L2.

**Bridge** — программный свитч:

```text
хост --- eth1 ---+
                 +--- br0 --- как один сегмент
туннель vxlan10 -+
```

Команды:

```bash
brctl addbr br0
brctl addif br0 vxlan10
brctl addif br0 eth1
ip link set br0 up
```

Перепутал eth0 и eth1 — туннель в underlay, а хост не в bridge. Конфиг «как в гайде», ping мёртвый.

---

## BUM — боль VXLAN без EVPN

**BUM** = Broadcast, Unknown unicast, Multicast.

В обычном свитче это «шли на все порты». В VXLAN «все порты» = все VTEP этого VNI. Нужно знать, **на какие IP** слать flood.

Part 2 даёт два учебных способа. Part 3 заменяет их control plane EVPN (Type 3).

### Static unicast (`remote`)

```bash
ip link add vxlan10 type vxlan id 10 remote 10.1.1.2 dstport 4789 dev eth0
```

`remote` — единственный сосед. BUM всегда на него. На двух VTEP идеально. На десятке — прописывать всех вручную.

### Dynamic multicast (`group`)

```bash
ip link add vxlan10 type vxlan id 10 group 239.1.1.1 dstport 4789 dev eth0
```

BUM на группу **239.1.1.1**. Кто подписался через IGMP — тот получает. Соседей заранее знать не надо.

Unicast, когда MAC уже в FDB, всё равно идёт точечно, не в группу.

Проверка:

```bash
ip maddr show
ip -d link show vxlan10
```

### EVPN (Part 3) — без remote и без group

```bash
ip link add vxlan10 type vxlan id 10 dstport 4789 local 1.1.1.2 nolearning
```

- `local` — outer source = loopback VTEP (стабильный адрес).
- нет `remote` / `group` — список VTEP берётся из BGP Type 3, MAC — из Type 2.
- `nolearning` — не учить MAC из туннеля, этим занимается EVPN.

---

## Путь ping в Part 2 (static)

1. Host1 ARP: кто 30.1.1.2? dst MAC ff:ff:ff:ff:ff:ff.
2. Фрейм → eth1 роутера1 → `br0` → `vxlan10`.
3. Инкапсуляция, outer dst = 10.1.1.2, UDP 4789, VNI 10.
4. Роутер2 снимает оболочку, `br0` → eth1 → host2.
5. ARP reply обратно тем же туннелем.
6. ICMP уже с известными MAC, снова внутри VXLAN.

На underlay `tcpdump -i eth0 -n port 4789` — видны UDP, не «голый» ICMP. ICMP спрятан внутри.

---

## VXLAN vs VLAN vs VPN «в быту»

- **VLAN** — тег в Ethernet, один физический свитч/кампус, мало ID.
- **VXLAN** — L2 поверх IP, много VNI, для DC.
- **«VPN»** в EVPN — не домашний OpenVPN, а *сервис*: клиентский L2/L3 через фабрику оператора. VXLAN здесь часто dataplane.

По заданию: EVPN **без MPLS**, чтобы упростить. Датаплейн — VXLAN, не MPLS-метки.

---


## Коротко

VXLAN (RFC 7348) инкапсулирует Ethernet в UDP 4789 и несёт его по IP. VNI 10 — номер сегмента, VTEP — концы туннеля, bridge склеивает туннель с портом хоста. Underlay — IP роутеров, overlay — сеть хостов. BUM в P2 — remote или multicast; в P3 список VTEP и MAC даёт EVPN, а VXLAN только возит фреймы.
