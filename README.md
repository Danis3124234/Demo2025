# demo2024

## Таблица
| Имя устройства | Интерфейс | IPv4/IPv6     | Маска/Префикс      | Шлюз          |
| -------------  | --------- | ------------- | ------------------ | ------------- | 
| ISP            | gi1/0/1   | DHCP          |                    | DHCP          |
|                | gi1/0/2   | 172.16.5.1    | 255.255.255.240/28 |               |
|                | gi1/0/3   | 172.16.4.1    | 255.255.255.240/28 |               |
| HQ-RTR         | ge0       | 172.16.4.2    | 255.255.255.240/28 | 172.16.4.1    |
|                | ge1.100   | 192.168.0.1   | 255.255.255.192/26 |               |
|                | ge1.200   | 192.168.0.65  | 255.255.255.240/28 |               |
|                | ge1.999   | 192.168.0.81  | 255.255.255.248/29 |               |
|                | tunnel.1  | 172.16.1.1    | 255.255.255.252/30 |               |
| BR-RTR         | gi1/0/1   | 172.16.5.2    | 255.255.255.240/28 | 172.16.5.1    |
|                | gi1/0/2   | 192.168.1.1   | 255.255.255.224/27 |               |
|                | gre1      | 172.16.1.2    | 255.255.255.252/30 |               |
| HQ-SRV         | ens 192   | 192.168.0.2   | 255.255.255.192/26 | 192.168.0.1   |
| HQ-CLI         | ens 192   | DHCP          | 255.255.255.240/28 | 192.168.0.65  |
| BR-SRV         | ens 192   | 192.168.1.2   | 255.255.255.224/27 | 192.168.1.1   |

## Настройка IP-адресов
### ISP
```
configure

int gi1/0/1
ip address dhcp
no shutdown

int gi1/0/3
ip address 172.16.4.1/28
ip firewall disable
no shutdown

int gi1/0/2
ip address 172.16.5.1/28
ip firewall disable
no shutdown

commit
confirm
```
### HQ-RTR (EcoRouter)
Создаем сущность интерфейса и назначаем IP

```
int TO-ISP
ip address 172.16.4.2/28
no shutdown
```

Привязываем созданный интерфейс к физическому протоколу

1. Заходим в `port ge0`
2. Создаем service-instance `service-instance SI-ISP`
3. Указываем, что кадры на этом интерфейсе будут без тега `encapsulation untagged`
4. Привязываем сущьность интрефейса к порту `connect ip interface TO-ISP`

```
port ge0
service-instance SI-ISP
encapsulation untagged
connect ip interface TO-ISP
```

Создаем интерфейсы, которые будут обрабатывать трафик vlan 100, 200, 999

```
interface HQ-SRV
ip mtu 1500
ip address 192.168.0.1/26
interface HQ-CLI
ip mtu 1500
ip address 192.168.0.65/28
interface HQ-MGMT
ip mtu 1500
ip address 192.168.0.81/29
```

Заходим на порт и создаем для каждой `vlan` свой `service-instance`.

1. Заходим в `port ge1`.
2. Создаем service-instance `service-instance ge1/vlan100`
3. Указываем инкапсуляцию для `100 vlan`
4. Чтобы кадры из этого интерфейса выходили с тегом задаем настройку `rewrite pop 1`
5. Привязываем сущьность интрефейса к порту `connect ip interface HQ-SRV`

Проделываем эти действия для vlan 200 и 999

```
port ge1
mtu 9234
service-instance ge1/vlan100
encapsulation dot1q 100
rewrite pop 1
connect ip interface HQ-SRV
service-instance ge1/vlan200
encapsulation dot1q 200
rewrite pop 1
connect ip interface HQ-CLI
service-instance ge1/vlan999
encapsulation dot1q 999
rewrite pop 1
connect ip interface HQ-MGMT
```

Создаем GRE туннель

```
interface tunnel.1
ip mtu 1400
ip address 172.16.1.1/30
ip tunnel 172.16.4.2 172.16.5.2 mode gre
```

Задаем маршрут по умолчанию в сторону ISP

```
ip route 0.0.0.0/0 172.16.4.1
```

### HQ-SRV

```
echo "TYPE=eth
DISABLED=no
NM_CONTROLLED=no
BOOTPROTO=static
CONFIG_IPv4=yes" > /etc/net/ifaces/ens192/options
```

```
echo 192.168.0.2/26 > /etc/net/ifaces/ens192/ipv4address
```

```
echo default via 192.168.0.1 > /etc/net/ifaces/ens192/ipv4route
```

```
systemctl restart network
```

### BR-RTR

```
configure
interface gigabitethernet 1/0/1
ip firewall disable
ip address 172.16.5.2/28
no shutdown
exit
interface gigabitethernet 1/0/2
ip firewall disable
ip address 192.168.1.1/27
no shutdown
exit
tunnel gre 1
ttl 16
mtu 1400
ip firewall disable
local address 172.16.5.2
remote address 172.16.4.2
ip address 172.16.1.2/30
enable
exit
ip route 0.0.0.0/0 172.16.5.1
#commit
#confirm
```

### BR-SRV

```
echo "TYPE=eth
DISABLED=no
NM_CONTROLLED=no
BOOTPROTO=static
CONFIG_IPv4=yes" > /etc/net/ifaces/ens192/options
```

```
echo 192.168.1.2/26 > /etc/net/ifaces/ens192/ipv4address
```

```
echo default via 192.168.1.1 > /etc/net/ifaces/ens192/ipv4route
```

```
echo nameserver 192.168.0.2 > /etc/net/ifaces/ens192/resolv.conf
```

```
systemctl restart network
```

```
ip address
```

## OSPF
### HQ-RTR
```
router ospf 1
 network 172.16.1.0 0.0.0.3 area 0.0.0.0
 network 192.168.0.0 0.0.0.255 area 0.0.0.0
!
interface tunnel.1
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 Demo2025
 ip ospf network point-to-point
```
### BR-RTR
```
key-chain ospfkey
key 1
key-string ascii-text Demo2025
exit
exit

router ospf 1
area 0.0.0.0
enable
exit
enable
exit

interface gigabitethernet 1/0/2
ip ospf instance 1
ip ospf
exit
tunnel gre 1
ip ospf instance 1
ip ospf network point-to-point
ip ospf authentication key-chain ospfkey
ip ospf authentication algorithm md5
ip ospf
exit

#commit
#confirm
```

### Проверка
```
sh ip ospf neighbor
```
## DHCP
```
ip pool HQ-NET200 1
range 192.168.0.66-192.168.0.70
!
dhcp-server 1
lease 86400
mask 255.255.255.0
pool HQ-NET200 1
dns 192.168.0.2
domain-name au-team.irpo
gateway 192.168.0.65
mask 255.255.255.240
!
interface HQ-CLI
dhcp-server 1
```
dns - адрес машины на которой dns сервер (HQ-SRV)
gateway, mask - ip адрес порта ge1.200 на HQ-RTR
Для проверки DHCP необходимо проверить получает ли HQ-CLI адрес
