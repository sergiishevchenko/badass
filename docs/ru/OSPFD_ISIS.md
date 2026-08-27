# Что такое OSPFD и IS-IS

## Простыми словами

В Part 1 требуется два сервиса маршрутизации рядом с BGPD:

- **OSPFD** — демон протокола **OSPF** (Open Shortest Path First);
- **IS-IS** — протокол **Intermediate System to Intermediate System**, в FRR это демон **`isisd`**.

Оба — **IGP** (Interior Gateway Protocol): протоколы *внутри* одной автономной системы / одного дата-центра. Они строят карту IP-достижимости между роутерами.

В проекте роли разные:

| | OSPF (`ospfd`) | IS-IS (`isisd`) |
|--|----------------|-----------------|
| Part 1 | должен быть **active** | должен быть **active** (движок IS-IS) |
| Part 3 | **реально настраиваем** underlay | обычно только запущен, без лаб-топологии |
| Зачем в проекте | loopback ↔ loopback, чтобы поднялся BGP | требование subject «IS-IS routing engine» |

Коротко: **ospfd делает underlay в P3**; **isisd нужен, чтобы формально закрыть Part 1** и уметь объяснить, что это за протокол.

---

# Часть 1 — OSPFD / OSPF

## Что такое OSPF

**OSPF** = **Open Shortest Path First** (RFC 2328 для OSPFv2 / IPv4).

Каждый роутер:

1. находит соседей (Hello);
2. обменивается описанием линков (LSA → LSDB);
3. считает кратчайшие пути (алгоритм Дейкстры / SPF);
4. кладёт лучшие маршруты в таблицу (через **zebra** → ядро).

В отличие от BGP, OSPF не про «Интернет между провайдерами». Он про «наши роутеры в одной сети быстро сходятся».

## Что такое ospfd

**ospfd** — демон OSPF в Zebra/Quagga/FRR. Отдельный процесс:

```text
ospfd  →  zebra  →  ядро Linux
```

- `ospfd` говорит OSPF с соседями и считает SPF;
- `zebra` ставит маршруты в kernel.

В `P1/daemons`:

```text
ospfd=yes
ospfd_options="  -A 127.0.0.1"
```

Проверка:

```bash
ps aux | grep ospfd
vtysh -c "show ip ospf"
vtysh -c "show ip ospf neighbor"
```

## Зачем OSPF в проекте (Part 3)

BGP-сессии у нас на **loopback** (`1.1.1.1` … `1.1.1.4`) с `update-source lo`.

Чтобы TCP 179 дошёл до чужого loopback, в underlay должен быть маршрут. Его даёт OSPF:

```text
router ospf
 network 10.1.1.0/30 area 0
 network 1.1.1.2/32 area 0
```

Без OSPF: loopback соседа не пингуется → BGP не Established → EVPN мёртв.  
С OSPF в состоянии **Full**: underlay зелёный → можно поднимать bgpd.

У нас с задании прямо сказана: *use OSPF to simplify the evaluation*.

## Важные слова OSPF

**Area** — зона. У нас всё в **area 0** (backbone). Для четырёх роутеров одной зоны достаточно.

**Neighbor / adjacency** — соседство. Смотришь `show ip ospf neighbor`.

**Full** — базы синхронизированы, всё ок. Если `Init`/`ExStart` зависли — линк, маска `/30`, area или интерфейс down.

**LSA / LSDB** — объявления о линках и общая база. На защите достаточно: «роутеры заливают карту сети и считают кратчайший путь».

**Hello** — периодические пакеты «я здесь». В tcpdump на underlay часто виден protocol OSPF (89), как в картинках задания.

**IGP** — класс протоколов (OSPF, IS-IS, RIP). OSPF — наш рабочий IGP.

## OSPF vs BGP в одной фразе

- **OSPF** — как доехать до `1.1.1.4` по физическим линкам.
- **BGP (bgpd)** — какие MAC/VNI объявить по EVPN (и классические Internet-маршруты в большом мире).

Они не дублируют друг друга в Part 3.

# Часть 2 — IS-IS и isisd

## Что такое IS-IS

**IS-IS** = **Intermediate System to Intermediate System**.

«Intermediate System» в терминологии ISO — по сути маршрутизатор. Протокол пришёл из мира ISO/CLNP, потом его научили носить IP (**Integrated IS-IS**). Сейчас это обычный IGP в больших сетях операторов, конкурент OSPF.

Идея та же, что у OSPF: соседи, база топологии, SPF, маршруты в ядро. Другая «родословная» и детали пакетов.

## Чем IS-IS отличается от OSPF

| | OSPF | IS-IS |
|--|------|-------|
| «Родной» мир | IP (IETF) | ISO, потом IP |
| Инкапсуляция | IP proto 89 | поверх L2 (не «просто IP packet» как OSPF) |
| Иерархия | areas (0 = backbone) | levels (L1/L2) |
| Популярность в DC/лабах | очень частый | чаще у ISP/backbone |
| В проекте | настраиваем | требуем демон |

## Что такое isisd

**isisd** — демон IS-IS в FRR/Quagga.

```text
isisd=yes
isisd_options="  -A 127.0.0.1"
```

Проверка Part 1:

```bash
ps aux | grep isisd
vtysh -c "show isis neighbor"
```

Если соседей нет — в нашей топологии это **нормально**: мы IS-IS между роутерами не настраивали. Важно, что **процесс есть** (routing engine service запущен).

# Общее: как это лежит в Part 1

Показать одной командой:

```bash
ps aux | grep -E "zebra|bgpd|ospfd|isisd"
```

Четыре процесса — Part 1 по демонам закрыт. Дальше Part 3 наполняет смыслом в основном **ospfd** + **bgpd**.

---

## Резюме

**OSPFD** — демон OSPF в FRR. OSPF — IGP: соседи, SPF, маршруты в ядро через zebra. В Part 3 он поднимает достижимость loopback, чтобы работал BGP.

**IS-IS** — другой IGP (ISO-корни, уровни L1/L2). В FRR это демон **isisd**. Задание требует, чтобы routing engine был активен; underlay в лабе мы строим на OSPF, как упрощение для оценки.
