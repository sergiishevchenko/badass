# Что такое BGPD

## Простыми словами

**bgpd** — это **демон BGP** в стеке Zebra / Quagga / FRRouting.

Расшифровка по частям:

- **BGP** — Border Gateway Protocol, протокол обмена маршрутами (тот, на котором держится Интернет и наш EVPN).
- **d** на конце — *daemon*, фоновый процесс. Не «включил BGP кнопкой в ядре», а отдельная программа, которая постоянно крутится и говорит с соседями.

Когда в Part 1 написано: *«The service BGPD active and configured»* — имеют в виду: процесс `bgpd` **запущен**, а не «уже настроен полный EVPN». Полную настройку BGP/EVPN вы делаете в Part 3 через `vtysh`. В Part 1 достаточно, что демон живой (часто уже с минимальным/дефолтным конфигом FRR).

В проекте `bgpd` — это тот кусок FRR, без которого не будет ни iBGP с Route Reflector, ни address-family `l2vpn evpn`.

---

## BGPD — не то же самое, что BGP

| | BGP | bgpd |
|--|-----|------|
| Что это | протокол (правила, сообщения, RFC) | программа, которая этот протокол реализует |
| Где живёт | «на бумаге» и на проводе (TCP 179) | процесс в Linux-контейнере |
| Аналогия | язык | человек, который на этом языке разговаривает |

Можно знать теорию BGP и при этом не запустить `bgpd` — соседей не будет. Можно запустить `bgpd` без `address-family l2vpn evpn` — сессия может подняться, а EVPN-маршрутов не будет.

---

## Где bgpd сидит в архитектуре FRR

```text
  ваш vtysh / конфиг
           |
         bgpd   ← считает BGP, держит сессии, EVPN NLRI
           |
         zebra  ← ставит то, что нужно, в ядро (и отдаёт интерфейсы)
           |
     ядро Linux
```

Важные роли:

- **bgpd** общается с другими BGP-роутерами (у нас — RR и VTEP).
- **zebra** не «говорит BGP по TCP 179». Он — посредник к интерфейсам и таблице ядра.
- **ospfd** — соседний демон для underlay; он не заменяет bgpd.

В `P1/daemons`:

```text
bgpd=yes
bgpd_options="   -A 127.0.0.1"
```

`-A 127.0.0.1` — управляющий сокет bgpd слушает localhost (к нему ходит vtysh). Сами BGP-сессии к соседям идут с интерфейсов/loopback, как вы настроите (`update-source lo` и т.д.).

---

## Что bgpd делает по шагам

1. Стартует вместе с FRR (`docker-start` читает `daemons`).
2. Читает конфиг (после `vtysh` / `write memory` — обычно `/etc/frr/frr.conf`).
3. Устанавливает TCP-сессии к `neighbor` (порт **179**).
4. Обменивается OPEN / KEEPALIVE / UPDATE.
5. Хранит BGP RIB (таблицу объявленных/принятых маршрутов).
6. Для обычного IPv4 мог бы отдавать префиксы в zebra → в kernel.
7. Для EVPN держит NLRI Type 2/Type 3 в семействе `l2vpn evpn` и стыкует это с VXLAN/EVPN-логикой FRR.

---

## BGPD в Part 1 vs Part 3

### Part 1 — «service BGPD active»

Проверяющий хочет увидеть процесс:

```bash
ps aux | grep bgpd
vtysh -c "show version"
```

Демон есть → требование «BGPD active» закрыто.  
«Configured» в Part 1 часто читают мягко: файл daemons включил bgpd, FRR поднял сервис. Жёсткая BGP-топология — в Part 3.

### Part 3 — bgpd уже «по делу»

На RR (`_sshevche-1`):

```text
router bgp 65000
 neighbor 1.1.1.2 remote-as 65000
 ...
 address-family l2vpn evpn
  neighbor 1.1.1.2 activate
  neighbor 1.1.1.2 route-reflector-client
```

На VTEP:

```text
router bgp 65000
 neighbor 1.1.1.1 remote-as 65000
 address-family l2vpn evpn
  neighbor 1.1.1.1 activate
  advertise-all-vni
```

Всё это конфигурация **процесса bgpd** (через vtysh). Без `bgpd=yes` эти команды некуда применять.

---

## Важные команды, которые относятся к bgpd

Смотреть глазами bgpd:

```bash
vtysh -c "show bgp summary"
vtysh -c "show bgp neighbors"
vtysh -c "show bgp l2vpn evpn"
vtysh -c "show bgp l2vpn evpn route type macip"
vtysh -c "show bgp l2vpn evpn route type multicast"
```

`show bgp summary` — сессии Established?  
`show bgp l2vpn evpn` — MP-BGP семейство живое, есть NLRI?

Процесс жив, а summary пустой — часто ещё не настроили `router bgp` / соседей / не поднялся OSPF до loopback.

---

## BGPD и соседние термины

- **BGP** — протокол.
- **bgpd** — демон, который его крутит.
- **MP-BGP** — режим/расширение: bgpd возит не только IPv4, но и `l2vpn evpn`.
- **EVPN** — смысл маршрутов (MAC, VNI); возит их bgpd.
- **NLRI** — содержимое UPDATE, которое bgpd объявляет и принимает.
- **zebra** — соседний демон; без него FRR-стек неполноценен.
- **ospfd** — underlay; bgpd у нас строится поверх достижимости loopback из OSPF.

Цепочка Part 3:

```text
ospfd → есть маршрут до 1.1.1.x
bgpd  → TCP 179 на loopback, EVPN UPDATE
VXLAN → фреймы хостов по UDP 4789
```

---

## Active and configured — как не попасть в ловушку

Part 1: BGPD **active and configured**, OSPFD тоже, плюс IS-IS service.

Практичный минимум:

1. В образе `bgpd=yes`.
2. После Start контейнера `ps` показывает `bgpd`.
3. `vtysh` отвечает (значит стек FRR сконфигурирован хотя бы до рабочего демона).

Для Part 3 «configured» уже значит реальные `neighbor`, `activate`, RR/VTEP роли.

Не путать: **IS-IS** у нас демон `isisd` запущен; отдельного «bgp для IS-IS» нет. BGPD ≠ isisd.

---

## Резюме

BGPD — демон BGP в FRR (наследник Zebra/Quagga). Он держит TCP-сессии с соседями и обрабатывает UPDATE, в том числе EVPN через MP-BGP. В Part 1 он должен быть запущен (`bgpd=yes`). В Part 3 мы его конфигурируем: AS 65000, соседи на loopback, `l2vpn evpn`, на RR — reflector clients, на VTEP — `advertise-all-vni`. Маршруты в underlay при этом даёт OSPF, а не bgpd.
