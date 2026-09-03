# BADASS — Полная инструкция по развёртыванию

## Что реально нужно

1. VM с Docker + GNS3
2. Собранные образы `dmiasnik-host` и `dmiasnik-router`
3. Правильно созданные шаблоны Docker в GNS3 (имя, image, adapters, console)
4. Три отдельные GNS3-топологии: P1, P2, P3
5. Применение конфигов из репозитория на каждое устройство
6. Проверка, что протоколы реально поднялись
7. Экспорт `P1.gns3project`, `P2.gns3project`, `P3.gns3project` **с образами**
8. Файлы конфигов рядом с экспортами в `P1/`, `P2/`, `P3/`

---

## 0. Требования к VM

Рекомендуется Ubuntu 22.04 / 24.04.

Минимум:
- 4 CPU
- 8 GB RAM
- 40 GB диск
- включённая виртуализация в BIOS (VT-x / AMD-V)

Если GNS3 запускается внутри VirtualBox/VMware, включи Nested Virtualization.

Проверка после входа в VM:

```bash
whoami
free -h
nproc
ls /dev/kvm
```

`/dev/kvm` желательно должен существовать.

---

## 1. Установка Docker

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y ca-certificates curl gnupg lsb-release
sudo apt install -y docker.io
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
```

Перелогинься или выполни:

```bash
newgrp docker
```

Проверка:

```bash
docker version
docker run --rm hello-world
```

Если `permission denied` на docker.sock — ты не в группе `docker` или не перелогинился.

---

## 2. Установка GNS3

```bash
sudo apt install -y software-properties-common
sudo add-apt-repository ppa:gns3/ppa
sudo apt update
sudo apt install -y gns3-gui gns3-server
```

Дополнительные пакеты, которые часто нужны:

```bash
sudo apt install -y \
  python3-pip \
  python3-pyqt5 \
  python3-pyqt5.qtsvg \
  python3-pyqt5.qtwebsockets \
  qemu-kvm \
  qemu-utils \
  libvirt-daemon-system \
  libvirt-clients \
  bridge-utils \
  wireshark
```

Группы пользователя:

```bash
sudo usermod -aG ubridge,libvirt,kvm,wireshark,docker $USER
```

Перезагрузи VM:

```bash
sudo reboot
```

После reboot проверь:

```bash
groups
gns3 --version
docker info
```

### Важно про IP forwarding

На хосте/VM:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-ipforward.conf
sudo sysctl --system
```

Проверка:

```bash
sysctl net.ipv4.ip_forward
# должно быть = 1
```

Без этого маршрутизация между контейнерами может вести себя странно.

---

## 3. Сборка Docker-образов

Из корня репозитория:

```bash
cd ~/badass   # или путь к твоему клону

docker build -t dmiasnik-host -f P1/Dockerfile.host P1/
docker build -t dmiasnik-router -f P1/Dockerfile.router P1/
```

Проверка образов:

```bash
docker images | grep dmiasnik
```

Проверка host:

```bash
docker run --rm dmiasnik-host sh -c "busybox | head -1; ip -V; ping -V; which tcpdump"
```

Проверка router:

```bash
docker run --rm -d --name test-router --privileged dmiasnik-router
sleep 4
docker exec test-router ps aux | grep -E "zebra|bgpd|ospfd|isisd|watchfrr"
docker exec test-router vtysh -c "show version"
docker exec test-router which brctl
docker exec test-router which tcpdump
docker stop test-router
```

Ожидаемо:
- видны `zebra`, `bgpd`, `ospfd`, `isisd`
- `vtysh` отвечает
- есть `brctl` и `tcpdump`

Если демоны не стартуют — образ собран неправильно или `daemons` не попал в `/etc/frr/daemons`.

---

## 4. Создание шаблонов Docker в GNS3

Открой GNS3:

`Edit → Preferences → Docker containers → New`

### 4.1 Шаблон host

Создай шаблон:

| Поле | Значение |
|------|----------|
| Image | `dmiasnik-host` |
| Name | `dmiasnik-host` |
| Adapters | `1` |
| Start command | оставить пустым или `sh` |
| Console type | `telnet` или `docker` |
| Network | bridge / default |

Практическая рекомендация:
- Console type: **`docker`** проще для shell
- Start command: пустой, чтобы сработал `CMD` из Dockerfile (`tail -f /dev/null`)

### 4.2 Шаблон router

Создай ещё один шаблон:

| Поле | Значение |
|------|----------|
| Image | `dmiasnik-router` |
| Name | `dmiasnik-router` |
| Adapters | `4` |
| Start command | пустой |
| Console type | `docker` или `telnet` |

Почему `4` adapters:
- P1: нужен 1
- P2: роутеру нужно 2 (`eth0` underlay, `eth1` к хосту)
- P3 RR: нужно 3 (`eth0`, `eth1`, `eth2`)
- P3 VTEP: нужно 2

Лучше сразу поставить 4 adapters на router-шаблон, чем потом ломать топологию.

### 4.3 Как называть устройства на сцене

Имена на сцене должны содержать логин:

**P1 / P2**
- `_dmiasnik-1_host`
- `_dmiasnik-2`
- `_dmiasnik-1_s` / `_dmiasnik-1_g`
- `_dmiasnik-2_s` / `_dmiasnik-2_g`

**P3**
- `_sshevche-1` ... `_sshevche-4`
- `_sshevche-1_host` ... `_sshevche-3_host`

Имя на сцене != имя Docker image.
Image остаётся `dmiasnik-host` / `dmiasnik-router`, а hostname/label в GNS3 меняется вручную.

---

## 5. Как правильно применять конфиги

Есть 3 рабочих способа.

### Способ A — копировать команды вручную в консоль

Самый простой для защиты:
1. Start node
2. Open console
3. Вставить команды из файла конфига

### Способ B — скопировать скрипт внутрь контейнера

Узнать container id:

```bash
docker ps
```

Скопировать:

```bash
docker cp P2/_dmiasnik-1_s <container_id>:/tmp/setup.sh
docker exec -it <container_id> sh /tmp/setup.sh
```

### Способ C — один раз открыть консоль и выполнить файл из репозитория построчно

На защите обычно ждут, что ты понимаешь команды, а не просто запускаешь скрипт вслепую.

---

## 6. Part 1 — пошагово

### Топология

```
_dmiasnik-1_host ---- _dmiasnik-2
        eth0              eth0
```

### Действия

1. New blank project: `P1`
2. Перетащить host и router
3. Переименовать устройства
4. Соединить `eth0 ↔ eth0`
5. Start all
6. Открыть консоли
7. Выполнить:

Host:
```bash
ip addr add 10.1.1.1/24 dev eth0
ip link set eth0 up
```

Router:
```bash
ip addr add 10.1.1.2/24 dev eth0
ip link set eth0 up
```

### Что должно получиться

На host:
```bash
ping -c 3 10.1.1.2
```

На router:
```bash
ps aux | grep -E "zebra|bgpd|ospfd|isisd"
vtysh -c "show version"
vtysh -c "show interface brief"
```

Если ping не идёт:
- интерфейсы не `up`
- соединены не те adapters
- адреса в разных подсетях
- контейнер не стартовал

### Экспорт P1

`File → Export portable project`

Обязательно:
- include base images / include images
- сохранить как `P1/P1.gns3project`

Проверка:
```bash
file P1/P1.gns3project
# Zip archive data
```

---

## 7. Part 2 — пошагово

### Топология

```
_dmiasnik-1_host (eth0)
        |
     (eth1)
_dmiasnik-1 (router)
     (eth0)
        |
     (eth0)
_dmiasnik-2 (router)
     (eth1)
        |
_dmiasnik-2_host (eth0)
```

Критично:
- underlay между роутерами = `eth0 ↔ eth0`
- хосты цепляются к `eth1` роутеров

Если перепутать `eth0/eth1`, VXLAN поднимется, но bridge не попадет на нужный порт и ping не пойдёт.

### 7.1 Static VXLAN

На `_dmiasnik-1` выполни `P2/_dmiasnik-1_s`  
На `_dmiasnik-2` выполни `P2/_dmiasnik-2_s`  
На хостах — `P2/_dmiasnik-1_host` и `P2/_dmiasnik-2_host`

Проверки на роутере:

```bash
ip addr show eth0
ip -d link show vxlan10
brctl show
bridge fdb show
```

Ожидаемо:
- `vxlan id 10`
- `remote <peer>`
- `dstport 4789`
- в `br0` есть `vxlan10` и `eth1`

На host1:
```bash
ping -c 5 30.1.1.2
```

На роутере во время ping:
```bash
tcpdump -i eth0 -n port 4789
```

Должны быть UDP VXLAN пакеты.

### 7.2 Dynamic Multicast VXLAN

Важно: не оставляй старый static VXLAN.

На обоих роутерах:

```bash
ip link set eth1 down || true
ip link set br0 down || true
brctl delif br0 eth1 || true
brctl delif br0 vxlan10 || true
ip link del vxlan10 || true
brctl delbr br0 || true
```

Потом применить `P2/_dmiasnik-1_g` и `P2/_dmiasnik-2_g`.

Проверки:

```bash
ip -d link show vxlan10
ip maddr show
bridge fdb show dev vxlan10
ping -c 5 30.1.1.2
```

Ожидаемо:
- в `vxlan10` есть `group 239.1.1.1`
- в multicast membership видна группа
- ping проходит

### Частая проблема multicast

Если static работает, а multicast нет:
1. Проверь, что оба роутера в `group 239.1.1.1`
2. Проверь underlay ping `10.1.1.1 ↔ 10.1.1.2`
3. Пересоздай VXLAN/bridge полностью
4. Убедись, что хосты снова `up` и имеют IP

Иногда удобнее сделать **два разных GNS3 project**:
- `P2-static`
- `P2-multicast`


### Экспорт P2

`File → Export portable project` → `P2/P2.gns3project`

---

## 8. Part 3 — пошагово

### Топология

```
                 _sshevche-1 (RR)
                /      |       \
               /       |        \
      _sshevche-2  _sshevche-3  _sshevche-4
           |           |            |
   _sshevche-1_host _sshevche-2_host _sshevche-3_host
```

Линки:

| From | Interface | To | Interface |
|------|-----------|----|-----------|
| `_sshevche-1` | eth0 | `_sshevche-2` | eth0 |
| `_sshevche-1` | eth1 | `_sshevche-3` | eth0 |
| `_sshevche-1` | eth2 | `_sshevche-4` | eth0 |
| `_sshevche-2` | eth1 | `_sshevche-1_host` | eth0 |
| `_sshevche-3` | eth1 | `_sshevche-2_host` | eth0 |
| `_sshevche-4` | eth1 | `_sshevche-3_host` | eth0 |

Перед стартом проверь, что у RR в GNS3 действительно 3+ adapters.

### Порядок настройки — критичен

1. Поднять все 4 роутера
2. Применить конфиги роутеров:
   - `_sshevche-1`
   - `_sshevche-2`
   - `_sshevche-3`
   - `_sshevche-4`
3. Подождать 15–30 секунд
4. Проверить OSPF
5. Проверить BGP
6. Проверить Type 3
7. Только потом поднимать хосты
8. Проверить Type 2
9. Делать ping

Если поднять хосты раньше, чем BGP EVPN, можно запутаться в отладке.

### Проверки underlay / OSPF

На RR:

```bash
vtysh -c "show ip ospf neighbor"
ping -c 2 1.1.1.2
ping -c 2 1.1.1.3
ping -c 2 1.1.1.4
```

Ожидаемо: соседи `Full`, loopback VTEP пингуются.

Если OSPF не Full:
- неправильные `/30`
- eth перепутаны
- `network ... area 0` не совпадает с реальными адресами
- интерфейсы не `up`

### Проверки BGP EVPN

На RR:

```bash
vtysh -c "show bgp summary"
vtysh -c "show bgp l2vpn evpn"
vtysh -c "show bgp l2vpn evpn route type multicast"
```

Ожидаемо:
- 3 peers `Established`
- есть Type 3 / IMET маршруты
- Type 2 ещё нет, если хосты не подняты

На VTEP:

```bash
ip -d link show vxlan10
brctl show
vtysh -c "show bgp summary"
vtysh -c "show bgp l2vpn evpn"
```

### Поднятие хостов

Применить:
- `P3/_sshevche-1_host`
- `P3/_sshevche-2_host`
- `P3/_sshevche-3_host`

Потом на RR/VTEP:

```bash
vtysh -c "show bgp l2vpn evpn route type macip"
vtysh -c "show evpn mac vni 10"
```

Ожидаемо: появляются Type 2 / MAC routes.

### Финальный ping

С `_sshevche-1_host`:

```bash
ping -c 5 20.1.1.2
ping -c 5 20.1.1.3
```

Во время ping на VTEP:

```bash
tcpdump -i eth0 -n port 4789
```

Должны быть VXLAN пакеты.

### Экспорт P3

`File → Export portable project` → `P3/P3.gns3project`
