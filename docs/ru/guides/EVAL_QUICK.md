# Защита BADASS

Пошаговая защита с объяснением каждого шага — в [`EVAL.md`](EVAL.md). Здесь коротко: откуда взять проекты GNS3 и полный список команд без пояснений.

---

## Архивы проектов

Проекты GNS3 лежат в intra-репозитории как portable project — файлы `.gns3project`. Это обычные ZIP-архивы, их экспортировали через **File → Export portable project**.

| Файл | Проект в GNS3 | Раздел ниже |
|------|---------------|-------------|
| `P1/P1.gns3project` | P1 | 1. Part 1 |
| `P2/P2.gns3project` | P2 static | 2. Part 2 static |
| `P2/P2-multicast.gns3project` | P2 multicast | 3. Part 2 multicast |
| `P3/P3.gns3project` | P3 | 4. Part 3 |

Если в intra файлы названы иначе, бери свои имена.

### Что в архиве есть

- узлы с именами, которые содержат логин (`dmiasnik-1-s`, `sshevche-1` и так далее);
- кабели между портами, например хост в `Ethernet1` роутера;
- число адаптеров у каждого узла;
- имя Docker-образа, из которого запускается узел.

### Чего в архиве нет

- **Docker-образов.** Узлы ссылаются на образы `dmiasnik-host` и `dmiasnik-router` по имени. На машине защиты их нужно собрать до импорта.
- **Настроек внутри контейнеров.** IP-адреса, `vxlan10`, `br0` и конфиг FRR живут только в запущенном контейнере. После **Start** узлы чистые: конфиг вводишь заново командами из раздела «Все команды подряд».

### Проверить архив

**Куда:** терминал Ubuntu, корень intra-репозитория.

```bash
file P1/P1.gns3project P2/P2.gns3project P2/P2-multicast.gns3project P3/P3.gns3project
```

**На экране:** у каждого файла `Zip archive data`.

---

## Подготовка машины

### 1. Склонировать intra-репозиторий

**Куда:** терминал Ubuntu.

```bash
git clone АДРЕС_INTRA_РЕПО badass
cd badass
```

**На экране:** `ls` показывает каталоги `P1`, `P2`, `P3`.

### 2. Собрать образы

**Куда:** терминал Ubuntu, корень репозитория. Нужен интернет.

```bash
docker build -t dmiasnik-host -f P1/Dockerfile.host P1/
docker build -t dmiasnik-router -f P1/Dockerfile.router P1/
docker images | grep dmiasnik
```

**На экране:** две строки, `dmiasnik-host` и `dmiasnik-router`. Имена должны совпадать в точности: архив ищет образы именно под этими тегами.

### 3. Импортировать проект

Один проект за раз. Перед импортом следующего в текущем нажми **Stop all**.

1. В GNS3 выбери **File → Import portable project…**
2. Укажи файл, например `P1/P1.gns3project`.
3. В окне импорта задай имя проекта (`P1`, `P2`, `P2-multicast`, `P3`) и нажми **OK**. Если проект с таким именем уже есть, GNS3 предложит его заменить — соглашайся.
4. Дождись, пока на холсте появится схема.
5. Нажми **Start all**. Для P3 сначала запусти только четыре роутера, хосты — позже, как в разделе 4.

**На экране:** иконки узлов стали запущенными. В терминале Ubuntu `docker ps` показывает контейнеры с IMAGE `dmiasnik-host` и `dmiasnik-router`.

Если узел не стартует и пишет, что образ не найден, вернись к шагу 2.

### 4. Ввести конфиг

Схема из архива уже соединена, но узлы пустые. Дальше команды из раздела ниже: найди ID через `docker ps`, зайди в контейнер через `docker exec -it ID sh` и вставь блок. В P3 вместо ручного ввода можно прогнать файл конфига:

```bash
docker cp P3/_sshevche-1 ID_sshevche-1:/tmp/setup.sh
docker exec -it ID_sshevche-1 sh /tmp/setup.sh
```

После **Stop all** настройки снова пропадают. После нового **Start** конфиг вводишь заново, архив при этом заново импортировать не нужно.

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
# Ubuntu, запущены только 4 роутера
docker ps --format 'table {{.ID}}\t{{.Image}}\t{{.Names}}'

# Ubuntu
docker exec -it ID_sshevche-1 sh
# внутри sshevche-1
ip addr add 10.1.1.1/30 dev eth0
ip addr add 10.1.1.5/30 dev eth1
ip addr add 10.1.1.9/30 dev eth2
ip link set eth0 up
ip link set eth1 up
ip link set eth2 up
ip addr add 1.1.1.1/32 dev lo
ip link set lo up
vtysh << 'EOF'
configure terminal
!
hostname _sshevche-1
!
router ospf
 network 10.1.1.0/30 area 0
 network 10.1.1.4/30 area 0
 network 10.1.1.8/30 area 0
 network 1.1.1.1/32 area 0
!
router bgp 65000
 bgp router-id 1.1.1.1
 no bgp default ipv4-unicast
 !
 neighbor 1.1.1.2 remote-as 65000
 neighbor 1.1.1.2 update-source lo
 neighbor 1.1.1.3 remote-as 65000
 neighbor 1.1.1.3 update-source lo
 neighbor 1.1.1.4 remote-as 65000
 neighbor 1.1.1.4 update-source lo
 !
 address-family l2vpn evpn
  neighbor 1.1.1.2 activate
  neighbor 1.1.1.2 route-reflector-client
  neighbor 1.1.1.3 activate
  neighbor 1.1.1.3 route-reflector-client
  neighbor 1.1.1.4 activate
  neighbor 1.1.1.4 route-reflector-client
 exit-address-family
!
end
write memory
EOF
exit

# Ubuntu
docker exec -it ID_sshevche-2 sh
# внутри sshevche-2
ip addr add 10.1.1.2/30 dev eth0
ip link set eth0 up
ip link set eth1 up
ip addr add 1.1.1.2/32 dev lo
ip link set lo up
ip link add vxlan10 type vxlan id 10 dstport 4789 local 1.1.1.2 nolearning
ip link set vxlan10 up
brctl addbr br0
brctl addif br0 vxlan10
brctl addif br0 eth1
ip link set br0 up
vtysh << 'EOF'
configure terminal
!
hostname _sshevche-2
!
router ospf
 network 10.1.1.0/30 area 0
 network 1.1.1.2/32 area 0
!
router bgp 65000
 bgp router-id 1.1.1.2
 no bgp default ipv4-unicast
 !
 neighbor 1.1.1.1 remote-as 65000
 neighbor 1.1.1.1 update-source lo
 !
 address-family l2vpn evpn
  neighbor 1.1.1.1 activate
  advertise-all-vni
 exit-address-family
!
end
write memory
EOF
exit

# Ubuntu
docker exec -it ID_sshevche-3 sh
# внутри sshevche-3
ip addr add 10.1.1.6/30 dev eth0
ip link set eth0 up
ip link set eth1 up
ip addr add 1.1.1.3/32 dev lo
ip link set lo up
ip link add vxlan10 type vxlan id 10 dstport 4789 local 1.1.1.3 nolearning
ip link set vxlan10 up
brctl addbr br0
brctl addif br0 vxlan10
brctl addif br0 eth1
ip link set br0 up
vtysh << 'EOF'
configure terminal
!
hostname _sshevche-3
!
router ospf
 network 10.1.1.4/30 area 0
 network 1.1.1.3/32 area 0
!
router bgp 65000
 bgp router-id 1.1.1.3
 no bgp default ipv4-unicast
 !
 neighbor 1.1.1.1 remote-as 65000
 neighbor 1.1.1.1 update-source lo
 !
 address-family l2vpn evpn
  neighbor 1.1.1.1 activate
  advertise-all-vni
 exit-address-family
!
end
write memory
EOF
exit

# Ubuntu
docker exec -it ID_sshevche-4 sh
# внутри sshevche-4
ip addr add 10.1.1.10/30 dev eth0
ip link set eth0 up
ip link set eth1 up
ip addr add 1.1.1.4/32 dev lo
ip link set lo up
ip link add vxlan10 type vxlan id 10 dstport 4789 local 1.1.1.4 nolearning
ip link set vxlan10 up
brctl addbr br0
brctl addif br0 vxlan10
brctl addif br0 eth1
ip link set br0 up
vtysh << 'EOF'
configure terminal
!
hostname _sshevche-4
!
router ospf
 network 10.1.1.8/30 area 0
 network 1.1.1.4/32 area 0
!
router bgp 65000
 bgp router-id 1.1.1.4
 no bgp default ipv4-unicast
 !
 neighbor 1.1.1.1 remote-as 65000
 neighbor 1.1.1.1 update-source lo
 !
 address-family l2vpn evpn
  neighbor 1.1.1.1 activate
  advertise-all-vni
 exit-address-family
!
end
write memory
EOF
exit

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
