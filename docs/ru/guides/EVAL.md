# Защита BADASS

Команда вводится только туда, куда сказано в шаге: в терминал Ubuntu или внутрь контейнера.

| Где ты | Какие команды сюда |
|---------|--------------------|
| Терминал Ubuntu, снаружи контейнера | `docker images`, `docker ps`, `docker exec`, `docker cp` |
| Внутри контейнера, приглашение `/ #` | `ip`, `ping`, `brctl`, `vtysh`, `tcpdump`, `ps` |

Окно **Console** в GNS3 не открывай. Туда команды не попадают.

Если на экране `/ #`, а следующий шаг начинается с `docker`, сначала введи `exit`. Ты снова в терминале Ubuntu. Контейнер при этом не гаснет.

`ip addr add` и `ip link` при успехе ничего не печатают: снова `/ #` и никакой ошибки. Это нормально. Результат смотри следующей командой `show` или `ping`.

Один проект GNS3 за раз. Перед следующим нажми в GNS3 **Stop all**. После Stop адреса пропадают: после нового **Start** конфиг вводишь заново.

Как найти ID контейнера. В терминале Ubuntu:

```bash
docker ps --format 'table {{.ID}}\t{{.Image}}\t{{.Names}}'
```

Так видны только запущенные контейнеры. Чтобы в тот же список попали и остановленные, добавь `-a` сразу после `ps`:

```bash
docker ps -a --format 'table {{.ID}}\t{{.Image}}\t{{.Names}}'
```

На экране ровно три колонки: `CONTAINER ID`, `IMAGE`, `NAMES`. ID — набор символов вроде `09f2819cace1`. В шагах ниже бери ID из списка без `-a`: там только то, что сейчас запущено в GNS3. Бери строку, у которой:

- IMAGE равен тому, что написано в шаге (`dmiasnik-host` или `dmiasnik-router`);
- NAMES содержит имя узла с холста (`sshevche-1`, `dmiasnik-1-s` и так далее).

Дальше в командах вместо `ID` подставь эти символы без скобок.

---

## 0. Образы

**Куда:** терминал Ubuntu. Внутри контейнера быть не должен. GNS3 не нужен.

```bash
docker images | grep dmiasnik
```

**На экране:** две строки, в них есть `dmiasnik-host` и `dmiasnik-router`. Если пусто — образы не собраны, к GNS3 не переходи.

---

## 1. Part 1

В GNS3 открой проект **P1** и нажми **Start all**. Подожди несколько секунд, иконки должны стать запущенными. В P1 два узла: один хост, один роутер.

### 1.1. Узнать ID

**Куда:** терминал Ubuntu.

```bash
docker ps --format 'table {{.ID}}\t{{.Image}}\t{{.Names}}'
```

**На экране:** две строки контейнеров.

- IMAGE `dmiasnik-host` — это хост. Запомни его ID, ниже он называется ID хоста.
- IMAGE `dmiasnik-router` — это роутер. Запомни его ID, ниже он называется ID роутера.

### 1.2. Адрес на хосте

**Куда:** тот же терминал Ubuntu.

```bash
docker exec -it ID_ХОСТА sh
```

**На экране:** приглашение сменилось на `/ #`. Следующие три команды вводятся здесь, не в Ubuntu.

```bash
ip addr add 10.1.1.1/24 dev eth0
ip link set eth0 up
ip addr show eth0
```

**На экране после `ip addr show eth0`:** строка `inet 10.1.1.1/24` и состояние `UP`. Первые две команды молчат.

Выйди наружу:

```bash
exit
```

**На экране:** снова терминал Ubuntu.

### 1.3. Адрес на роутере

**Куда:** терминал Ubuntu.

```bash
docker exec -it ID_РОУТЕРА sh
```

**На экране:** `/ #`.

```bash
ip addr add 10.1.1.2/24 dev eth0
ip link set eth0 up
ip addr show eth0
```

**На экране после `ip addr show eth0`:** строка `inet 10.1.1.2/24` и `UP`.

Останься в этом `/ #`. Проверка роутера идёт здесь же.

### 1.4. Проверка роутера

**Куда:** тот же `/ #` роутера. Не выходи.

```bash
ping -c 3 10.1.1.1
ps aux | grep -E "zebra|bgpd|ospfd|isisd"
vtysh -c "show version"
```

**На экране:**

- `ping`: три строки `64 bytes from 10.1.1.1` и `0% packet loss`.
- `ps`: отдельные строки процессов `zebra`, `bgpd`, `ospfd`, `isisd`.
- `vtysh`: может быть строка `Can't open configuration file /etc/frr/vtysh.conf`. Ниже обязательно текст `FRRouting` и номер версии. Это ответ команды.

```bash
exit
```

**На экране:** терминал Ubuntu.

### 1.5. Проверка хоста

**Куда:** терминал Ubuntu.

```bash
docker exec -it ID_ХОСТА sh
```

**На экране:** `/ #` хоста.

```bash
ping -c 3 10.1.1.2
```

**На экране:** три строки `64 bytes from 10.1.1.2` и `0% packet loss`.

```bash
exit
```

В GNS3 нажми **Stop all**.

---

## 2. Part 2 static

В GNS3 открой проект **P2** и нажми **Start all**. Четыре узла:

```text
dmiasnik-1-host —— dmiasnik-1-s ====== dmiasnik-2-s —— dmiasnik-2-host
              eth1              eth0              eth1
```

### 2.1. Узнать четыре ID

**Куда:** терминал Ubuntu. Если видишь `/ #`, сначала `exit`.

```bash
docker ps --format 'table {{.ID}}\t{{.Image}}\t{{.Names}}'
```

**На экране:** четыре контейнера. Сопоставь NAMES и запиши ID:

| NAMES содержит | Кто это |
|----------------|---------|
| `dmiasnik-1-host` | левый хост |
| `dmiasnik-1-s` | левый роутер |
| `dmiasnik-2-s` | правый роутер |
| `dmiasnik-2-host` | правый хост |

IMAGE у хостов `dmiasnik-host`, у роутеров `dmiasnik-router`.

### 2.2. Левый роутер

**Куда:** терминал Ubuntu.

```bash
docker exec -it ID_ЛЕВОГО_РОУТЕРА sh
```

**На экране:** `/ #`. Вставь блок целиком.

```bash
ip addr add 10.1.1.1/24 dev eth0
ip link set eth0 up
ip link add vxlan10 type vxlan id 10 remote 10.1.1.2 dstport 4789 dev eth0
ip link set vxlan10 up
brctl addbr br0
brctl addif br0 vxlan10
brctl addif br0 eth1
ip link set br0 up
ip link set eth1 up
```

**На экране:** команды молчат, ошибок нет, приглашение `/ #`.

Проверка в этом же `/ #`:

```bash
ip -d link show vxlan10
brctl show
```

**На экране:**

- у `vxlan10` есть `id 10`, `remote 10.1.1.2`, `dstport 4789`;
- `brctl show` показывает мост `br0`, в нём интерфейсы `vxlan10` и `eth1`.

```bash
exit
```

### 2.3. Правый роутер

**Куда:** терминал Ubuntu.

```bash
docker exec -it ID_ПРАВОГО_РОУТЕРА sh
```

**На экране:** `/ #`.

```bash
ip addr add 10.1.1.2/24 dev eth0
ip link set eth0 up
ip link add vxlan10 type vxlan id 10 remote 10.1.1.1 dstport 4789 dev eth0
ip link set vxlan10 up
brctl addbr br0
brctl addif br0 vxlan10
brctl addif br0 eth1
ip link set br0 up
ip link set eth1 up
```

**На экране:** молчание без ошибок.

В этом же `/ #`:

```bash
ip -d link show vxlan10
brctl show
```

**На экране:** `id 10`, `remote 10.1.1.1`, `dstport 4789`. В `br0` есть `vxlan10` и `eth1`.

```bash
exit
```

### 2.4. Левый хост

**Куда:** терминал Ubuntu.

```bash
docker exec -it ID_ЛЕВОГО_ХОСТА sh
```

**На экране:** `/ #`.

```bash
ip addr add 30.1.1.1/24 dev eth0
ip link set eth0 up
ip addr show eth0
```

**На экране после `show`:** `inet 30.1.1.1/24` и `UP`.

```bash
exit
```

### 2.5. Правый хост

**Куда:** терминал Ubuntu.

```bash
docker exec -it ID_ПРАВОГО_ХОСТА sh
```

**На экране:** `/ #`.

```bash
ip addr add 30.1.1.2/24 dev eth0
ip link set eth0 up
ip addr show eth0
```

**На экране после `show`:** `inet 30.1.1.2/24` и `UP`.

```bash
exit
```

### 2.6. Ping

**Куда:** терминал Ubuntu, затем внутрь левого хоста.

```bash
docker exec -it ID_ЛЕВОГО_ХОСТА sh
```

**На экране:** `/ #`.

```bash
ping -c 5 30.1.1.2
```

**На экране:** пять строк `64 bytes from 30.1.1.2` и `0% packet loss`.

```bash
exit
```

### 2.7. Захват туннеля

**Куда:** терминал Ubuntu.

```bash
docker exec -it ID_ЛЕВОГО_РОУТЕРА sh
```

**На экране:** `/ #` левого роутера.

```bash
tcpdump -i eth0 -n port 4789
```

**На экране:** `tcpdump: listening on eth0`. Строк с пакетами пока нет. Окно не закрывай.

Открой **второй** терминал Ubuntu. В нём:

```bash
docker exec -it ID_ЛЕВОГО_ХОСТА sh
ping -c 5 30.1.1.2
```

**На экране первого терминала** (там, где `tcpdump`): строки `UDP` с портом `4789`.

Остановить захват: в терминале с `tcpdump` нажми Ctrl+C, затем `exit`. Во втором терминале тоже `exit`.

В GNS3 нажми **Stop all**.

---

## 3. Part 2 multicast

В GNS3 открой проект **P2-multicast** и нажми **Start all**. Кабели те же. Имена роутеров другие: `dmiasnik-1-g` и `dmiasnik-2-g`. Хосты снова `dmiasnik-1-host` и `dmiasnik-2-host`.

### 3.1. Узнать четыре ID

**Куда:** терминал Ubuntu.

```bash
docker ps --format 'table {{.ID}}\t{{.Image}}\t{{.Names}}'
```

**На экране:** четыре контейнера. ID по NAMES:

| NAMES содержит | Кто это |
|----------------|---------|
| `dmiasnik-1-host` | левый хост |
| `dmiasnik-1-g` | левый роутер |
| `dmiasnik-2-g` | правый роутер |
| `dmiasnik-2-host` | правый хост |

### 3.2. Левый роутер

**Куда:** терминал Ubuntu.

```bash
docker exec -it ID_ЛЕВОГО_РОУТЕРА sh
```

**На экране:** `/ #`.

```bash
ip addr add 10.1.1.1/24 dev eth0
ip link set eth0 up
ip link add vxlan10 type vxlan id 10 group 239.1.1.1 dstport 4789 dev eth0
ip link set vxlan10 up
brctl addbr br0
brctl addif br0 vxlan10
brctl addif br0 eth1
ip link set br0 up
ip link set eth1 up
```

**На экране:** молчание без ошибок.

В этом же `/ #`:

```bash
ip -d link show vxlan10
ip maddr show dev eth0
brctl show
```

**На экране:**

- у `vxlan10` есть `id 10`, `group 239.1.1.1`, `dstport 4789`. Слова `remote` нет;
- в `ip maddr` на `eth0` есть `239.1.1.1`;
- в `br0` есть `vxlan10` и `eth1`.

```bash
exit
```

### 3.3. Правый роутер

**Куда:** терминал Ubuntu.

```bash
docker exec -it ID_ПРАВОГО_РОУТЕРА sh
```

**На экране:** `/ #`.

```bash
ip addr add 10.1.1.2/24 dev eth0
ip link set eth0 up
ip link add vxlan10 type vxlan id 10 group 239.1.1.1 dstport 4789 dev eth0
ip link set vxlan10 up
brctl addbr br0
brctl addif br0 vxlan10
brctl addif br0 eth1
ip link set br0 up
ip link set eth1 up
```

**На экране:** молчание без ошибок.

В этом же `/ #` те же три проверки:

```bash
ip -d link show vxlan10
ip maddr show dev eth0
brctl show
```

**На экране:** `group 239.1.1.1`, группа `239.1.1.1` в `ip maddr`, в `br0` порты `vxlan10` и `eth1`.

```bash
exit
```

### 3.4. Хосты

**Куда:** терминал Ubuntu, левый хост.

```bash
docker exec -it ID_ЛЕВОГО_ХОСТА sh
```

```bash
ip addr add 30.1.1.1/24 dev eth0
ip link set eth0 up
ip addr show eth0
```

**На экране после `show`:** `inet 30.1.1.1/24`.

```bash
exit
```

**Куда:** снова терминал Ubuntu, правый хост.

```bash
docker exec -it ID_ПРАВОГО_ХОСТА sh
```

```bash
ip addr add 30.1.1.2/24 dev eth0
ip link set eth0 up
ip addr show eth0
```

**На экране после `show`:** `inet 30.1.1.2/24`.

```bash
exit
```

### 3.5. Ping

**Куда:** терминал Ubuntu.

```bash
docker exec -it ID_ЛЕВОГО_ХОСТА sh
```

**На экране:** `/ #`.

```bash
ping -c 5 30.1.1.2
```

**На экране:** пять ответов `64 bytes from 30.1.1.2`, `0% packet loss`.

```bash
exit
```

В GNS3 нажми **Stop all**.

---

## 4. Part 3

В GNS3 открой проект **P3**. Схема:

```text
                 sshevche-1          верхний роутер, RR
                /     |      \
      sshevche-2   sshevche-3   sshevche-4
            |           |            |
    sshevche-1-host  sshevche-2-host  sshevche-3-host
       20.1.1.1        20.1.1.2         20.1.1.3
```

Сначала **Start** только четырёх роутеров. Три хоста не запускай, пока в шаге 4.4 не увидишь Type 3.

Команды `docker cp` и `docker ps --format ...` вводятся в терминале Ubuntu из папки репозитория `badass`. Если приглашение `/ #`, они не сработают: сначала `exit`.

### 4.1. Узнать ID роутеров

**Куда:** терминал Ubuntu, текущая папка — корень `badass` (`ls` показывает каталоги `P1`, `P2`, `P3`).

```bash
docker ps --format 'table {{.ID}}\t{{.Image}}\t{{.Names}}'
```

**На экране:** четыре контейнера, IMAGE у всех `dmiasnik-router`. ID по NAMES:

| NAMES содержит | Кто это |
|----------------|---------|
| `sshevche-1` | верхний роутер. Не перепутай с `sshevche-1-host`: хост ещё не запущен, его в списке нет |
| `sshevche-2` | левый средний роутер |
| `sshevche-3` | центральный средний роутер |
| `sshevche-4` | правый средний роутер |

### 4.2. Прогнать конфиг каждого роутера

**Куда:** терминал Ubuntu. Каждая пара команд — снаружи, не внутри `/ #`. Между парами `exit` не нужен: `docker exec ... sh /tmp/setup.sh` сам заканчивается и возвращает в терминал Ubuntu.

```bash
docker cp P3/_sshevche-1 ID_sshevche-1:/tmp/setup.sh
docker exec -it ID_sshevche-1 sh /tmp/setup.sh
```

**На экране:** приглашение на секунду может стать `/ #` или `sshevche-1#`, затем снова терминал Ubuntu. Ошибки `Error` быть не должно. Строка про `vtysh.conf` допустима.

```bash
docker cp P3/_sshevche-2 ID_sshevche-2:/tmp/setup.sh
docker exec -it ID_sshevche-2 sh /tmp/setup.sh
```

```bash
docker cp P3/_sshevche-3 ID_sshevche-3:/tmp/setup.sh
docker exec -it ID_sshevche-3 sh /tmp/setup.sh
```

```bash
docker cp P3/_sshevche-4 ID_sshevche-4:/tmp/setup.sh
docker exec -it ID_sshevche-4 sh /tmp/setup.sh
```

Подожди 15–30 секунд. Хосты всё ещё выключены.

### 4.3. Зайти на верхний роутер

**Куда:** терминал Ubuntu.

```bash
docker exec -it ID_sshevche-1 sh
```

**На экране:** `/ #`. Все команды шага 4.4 вводятся здесь. Это не `sshevche-2`, не `sshevche-3` и не `sshevche-4`.

### 4.4. OSPF, ping, BGP, Type 3

**Куда:** `/ #` контейнера `sshevche-1`.

```bash
vtysh -c "show ip ospf neighbor"
```

**На экране:** три строки с адресами `1.1.1.2`, `1.1.1.3`, `1.1.1.4`. В колонке состояния `Full`.

```bash
ping -c 2 1.1.1.2
ping -c 2 1.1.1.3
ping -c 2 1.1.1.4
```

**На экране:** у каждого ping две строки `64 bytes from` и `0% packet loss`.

```bash
vtysh -c "show bgp summary"
```

**На экране:** три соседа `1.1.1.2`, `1.1.1.3`, `1.1.1.4`. Состояние `Estab` (или число вместо слова `Estab`, это тоже установленная сессия).

```bash
vtysh -c "show bgp l2vpn evpn route type multicast"
```

**На экране:** маршруты с пометкой `Type 3` или `[3]`. Пустой вывод — рано: подожди ещё и повтори эту команду.

Из этого `/ #` не выходи, если удобно держать роутер открытым. Для хостов нужен терминал Ubuntu: либо `exit`, либо второй терминал.

### 4.5. Запустить хосты и выдать им адреса

В GNS3 нажми **Start** на трёх хостах: `sshevche-1-host`, `sshevche-2-host`, `sshevche-3-host`.

**Куда:** терминал Ubuntu.

```bash
docker ps --format 'table {{.ID}}\t{{.Image}}\t{{.Names}}'
```

**На экране:** к четырём роутерам добавились три хоста, IMAGE `dmiasnik-host`. Запиши ID по NAMES.

| NAMES содержит | Куда воткнут | Адрес |
|----------------|--------------|-------|
| `sshevche-1-host` | под роутером `sshevche-2` | `20.1.1.1` |
| `sshevche-2-host` | под роутером `sshevche-3` | `20.1.1.2` |
| `sshevche-3-host` | под роутером `sshevche-4` | `20.1.1.3` |

**Куда:** терминал Ubuntu, хост `sshevche-1-host`.

```bash
docker exec -it ID_sshevche-1-host sh
```

**На экране:** `/ #`.

```bash
ip addr add 20.1.1.1/24 dev eth0
ip link set eth0 up
ip addr show eth0
```

**На экране после `show`:** `inet 20.1.1.1/24`.

```bash
exit
```

**Куда:** терминал Ubuntu, хост `sshevche-2-host`.

```bash
docker exec -it ID_sshevche-2-host sh
```

```bash
ip addr add 20.1.1.2/24 dev eth0
ip link set eth0 up
ip addr show eth0
```

**На экране после `show`:** `inet 20.1.1.2/24`.

```bash
exit
```

**Куда:** терминал Ubuntu, хост `sshevche-3-host`.

```bash
docker exec -it ID_sshevche-3-host sh
```

```bash
ip addr add 20.1.1.3/24 dev eth0
ip link set eth0 up
ip addr show eth0
```

**На экране после `show`:** `inet 20.1.1.3/24`.

```bash
exit
```

### 4.6. Type 2 на верхнем роутере

**Куда:** терминал Ubuntu.

```bash
docker exec -it ID_sshevche-1 sh
```

**На экране:** `/ #` именно `sshevche-1`.

```bash
vtysh -c "show bgp l2vpn evpn route type macip"
vtysh -c "show evpn mac vni 10"
```

**На экране:**

- первая команда больше не пустая: есть маршруты Type 2 (`[2]` или слова `MAC/IP`);
- вторая показывает MAC-адреса на VNI 10.

```bash
exit
```

### 4.7. Ping между хостами

**Куда:** терминал Ubuntu.

```bash
docker exec -it ID_sshevche-1-host sh
```

**На экране:** `/ #` хоста с адресом `20.1.1.1`.

```bash
ping -c 5 20.1.1.2
ping -c 5 20.1.1.3
```

**На экране:** оба ping дают ответы `64 bytes from` и `0% packet loss`.

```bash
exit
```

### 4.8. Захват VXLAN

**Куда:** терминал Ubuntu, роутер `sshevche-2` (левый средний, не верхний и не хост).

```bash
docker exec -it ID_sshevche-2 sh
```

**На экране:** `/ #`.

```bash
tcpdump -i eth0 -n port 4789
```

**На экране:** `listening on eth0`.

Второй терминал Ubuntu:

```bash
docker exec -it ID_sshevche-1-host sh
ping -c 5 20.1.1.3
```

**На экране терминала с `tcpdump`:** строки UDP, порт `4789`.

Ctrl+C в терминале с `tcpdump`, затем `exit`. Во втором терминале `exit`.

---

## Все команды подряд

`ID_...` замени на ID из `docker ps`.

### 0. Образы

```bash
# Ubuntu
docker images | grep dmiasnik
```

### 1. Part 1

```bash
# Ubuntu
docker ps --format 'table {{.ID}}\t{{.Image}}\t{{.Names}}'

# Ubuntu
docker exec -it ID_ХОСТА sh
# внутри хоста
ip addr add 10.1.1.1/24 dev eth0
ip link set eth0 up
ip addr show eth0
exit

# Ubuntu
docker exec -it ID_РОУТЕРА sh
# внутри роутера
ip addr add 10.1.1.2/24 dev eth0
ip link set eth0 up
ip addr show eth0
ping -c 3 10.1.1.1
ps aux | grep -E "zebra|bgpd|ospfd|isisd"
vtysh -c "show version"
exit

# Ubuntu
docker exec -it ID_ХОСТА sh
# внутри хоста
ping -c 3 10.1.1.2
exit
```

### 2. Part 2 static

```bash
# Ubuntu
docker ps --format 'table {{.ID}}\t{{.Image}}\t{{.Names}}'

# Ubuntu
docker exec -it ID_ЛЕВОГО_РОУТЕРА sh
# внутри dmiasnik-1-s
ip addr add 10.1.1.1/24 dev eth0
ip link set eth0 up
ip link add vxlan10 type vxlan id 10 remote 10.1.1.2 dstport 4789 dev eth0
ip link set vxlan10 up
brctl addbr br0
brctl addif br0 vxlan10
brctl addif br0 eth1
ip link set br0 up
ip link set eth1 up
ip -d link show vxlan10
brctl show
exit

# Ubuntu
docker exec -it ID_ПРАВОГО_РОУТЕРА sh
# внутри dmiasnik-2-s
ip addr add 10.1.1.2/24 dev eth0
ip link set eth0 up
ip link add vxlan10 type vxlan id 10 remote 10.1.1.1 dstport 4789 dev eth0
ip link set vxlan10 up
brctl addbr br0
brctl addif br0 vxlan10
brctl addif br0 eth1
ip link set br0 up
ip link set eth1 up
ip -d link show vxlan10
brctl show
exit

# Ubuntu
docker exec -it ID_ЛЕВОГО_ХОСТА sh
# внутри dmiasnik-1-host
ip addr add 30.1.1.1/24 dev eth0
ip link set eth0 up
ip addr show eth0
exit

# Ubuntu
docker exec -it ID_ПРАВОГО_ХОСТА sh
# внутри dmiasnik-2-host
ip addr add 30.1.1.2/24 dev eth0
ip link set eth0 up
ip addr show eth0
exit

# Ubuntu
docker exec -it ID_ЛЕВОГО_ХОСТА sh
# внутри dmiasnik-1-host
ping -c 5 30.1.1.2
exit

# первый терминал Ubuntu
docker exec -it ID_ЛЕВОГО_РОУТЕРА sh
# внутри dmiasnik-1-s, окно не закрывать
tcpdump -i eth0 -n port 4789

# второй терминал Ubuntu
docker exec -it ID_ЛЕВОГО_ХОСТА sh
# внутри dmiasnik-1-host
ping -c 5 30.1.1.2
exit

# первый терминал: Ctrl+C, затем
exit
```

### 3. Part 2 multicast

```bash
# Ubuntu
docker ps --format 'table {{.ID}}\t{{.Image}}\t{{.Names}}'

# Ubuntu
docker exec -it ID_ЛЕВОГО_РОУТЕРА sh
# внутри dmiasnik-1-g
ip addr add 10.1.1.1/24 dev eth0
ip link set eth0 up
ip link add vxlan10 type vxlan id 10 group 239.1.1.1 dstport 4789 dev eth0
ip link set vxlan10 up
brctl addbr br0
brctl addif br0 vxlan10
brctl addif br0 eth1
ip link set br0 up
ip link set eth1 up
ip -d link show vxlan10
ip maddr show dev eth0
brctl show
exit

# Ubuntu
docker exec -it ID_ПРАВОГО_РОУТЕРА sh
# внутри dmiasnik-2-g
ip addr add 10.1.1.2/24 dev eth0
ip link set eth0 up
ip link add vxlan10 type vxlan id 10 group 239.1.1.1 dstport 4789 dev eth0
ip link set vxlan10 up
brctl addbr br0
brctl addif br0 vxlan10
brctl addif br0 eth1
ip link set br0 up
ip link set eth1 up
ip -d link show vxlan10
ip maddr show dev eth0
brctl show
exit

# Ubuntu
docker exec -it ID_ЛЕВОГО_ХОСТА sh
# внутри dmiasnik-1-host
ip addr add 30.1.1.1/24 dev eth0
ip link set eth0 up
ip addr show eth0
exit

# Ubuntu
docker exec -it ID_ПРАВОГО_ХОСТА sh
# внутри dmiasnik-2-host
ip addr add 30.1.1.2/24 dev eth0
ip link set eth0 up
ip addr show eth0
exit

# Ubuntu
docker exec -it ID_ЛЕВОГО_ХОСТА sh
# внутри dmiasnik-1-host
ping -c 5 30.1.1.2
exit
```

### 4. Part 3

```bash
# Ubuntu, папка badass, запущены только 4 роутера
docker ps --format 'table {{.ID}}\t{{.Image}}\t{{.Names}}'

# Ubuntu, папка badass
docker cp P3/_sshevche-1 ID_sshevche-1:/tmp/setup.sh
docker exec -it ID_sshevche-1 sh /tmp/setup.sh
docker cp P3/_sshevche-2 ID_sshevche-2:/tmp/setup.sh
docker exec -it ID_sshevche-2 sh /tmp/setup.sh
docker cp P3/_sshevche-3 ID_sshevche-3:/tmp/setup.sh
docker exec -it ID_sshevche-3 sh /tmp/setup.sh
docker cp P3/_sshevche-4 ID_sshevche-4:/tmp/setup.sh
docker exec -it ID_sshevche-4 sh /tmp/setup.sh

# Ubuntu, через 15–30 секунд
docker exec -it ID_sshevche-1 sh
# внутри sshevche-1
vtysh -c "show ip ospf neighbor"
ping -c 2 1.1.1.2
ping -c 2 1.1.1.3
ping -c 2 1.1.1.4
vtysh -c "show bgp summary"
vtysh -c "show bgp l2vpn evpn route type multicast"
exit

# Ubuntu, после Start трёх хостов в GNS3
docker ps --format 'table {{.ID}}\t{{.Image}}\t{{.Names}}'

# Ubuntu
docker exec -it ID_sshevche-1-host sh
# внутри sshevche-1-host
ip addr add 20.1.1.1/24 dev eth0
ip link set eth0 up
ip addr show eth0
exit

# Ubuntu
docker exec -it ID_sshevche-2-host sh
# внутри sshevche-2-host
ip addr add 20.1.1.2/24 dev eth0
ip link set eth0 up
ip addr show eth0
exit

# Ubuntu
docker exec -it ID_sshevche-3-host sh
# внутри sshevche-3-host
ip addr add 20.1.1.3/24 dev eth0
ip link set eth0 up
ip addr show eth0
exit

# Ubuntu
docker exec -it ID_sshevche-1 sh
# внутри sshevche-1
vtysh -c "show bgp l2vpn evpn route type macip"
vtysh -c "show evpn mac vni 10"
exit

# Ubuntu
docker exec -it ID_sshevche-1-host sh
# внутри sshevche-1-host
ping -c 5 20.1.1.2
ping -c 5 20.1.1.3
exit

# первый терминал Ubuntu
docker exec -it ID_sshevche-2 sh
# внутри sshevche-2, окно не закрывать
tcpdump -i eth0 -n port 4789

# второй терминал Ubuntu
docker exec -it ID_sshevche-1-host sh
# внутри sshevche-1-host
ping -c 5 20.1.1.3
exit

# первый терминал: Ctrl+C, затем
exit
```
