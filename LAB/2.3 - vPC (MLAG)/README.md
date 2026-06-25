# VXLAN. Multihoming

## Цель работы 

Настроить отказоустойчивое подключение клиентов с использованием EVPN Multihoming.



## План работы

- Подключить клиентов 2-я линками к различным Leaf
- Настроить агрегированный канал со стороны клиента
- Настроить multihoming для работы в Overlay сети.



## Тестовая среда

Схема сети в [PnetLab](..)

Конфигурации сетевых элементов

**LEAF:**
[DC1-SW3-LEAF1](../)
[DC1-SW3-LEAF2](../)
[DC1-SW3-LEAF3](../)
[DC1-SW3-LEAF4](../)

**SPINE:**
[DC1-SW3-SPINE1](../)
[DC1-SW3-SPINE2](../)

**CE:**
[DC1-SW3-CE1](../)
[DC1-SW3-CE2](../)



## Описание лабораторного стенда

![Схема сети DC1](../)

LEAF`ы разделены на кластеры отказоустойчивости по 2 коммутатора:

**DC1-SW3-LEAF1,2** - MLAG 

**DC1-SW3-LEAF3,4** - EVPN multihoming

Клиент (CST1) сымитирован коммутаторами (DC1-SW3-CE1,2) на которых UPLINK - это пара агрегированных линков c LACP  (каждый в сторону своего LEAF кластера)

Для разделения хостов клиента с разными услугами (SVC1,2) внутри коммутатора настроены разные VRF (100,200)



## Учет ID

VNI = порядковые значения
RD = loopback + порядковый id
RT = SPINE_AS + порядковый id

| VNI id | Service | Description |
| --- | --- | --- |
| 1 | [cst:1][svc:1] | vlan 100 |
| 2 | [cst:1][svc:2] | vlan 200 |
| 3 | [cst:1]#SYMMETRIC-IRB# | vrf CST1 |

| RD | RT | Service | Description |
| --- | --- | --- | --- |
| 1 | 1 | [cst:1][svc:1][svc:2] | vlan-aware-bundle CST1 |
| 2 | 2 | [cst:1]#SYMMETRIC-IRB# | vrf CST1 |



# Настройка

## MLAG пара

> на примере DC1-SW3-LEAF1 (DC1-SW3-LEAF2 настраивается аналогично)

### MLAG

для KEEPALIVE - отдельный физический линк между LEAF

для PEER-LINK - SVI VLAN 4000, выключим для него STP

```
!
vlan 4000
   name [cvlan:4000][tech:MLAG]#PEERVLAN
   trunk group PEERLINK
!
no spanning-tree vlan-id 4000
!
interface Vlan4000
   description [cvlan:4000][tech:MLAG]#MLAG_PEER-ADDRESS#
   ip address 172.16.0.1/31
!
interface Ethernet5
   description [dev:DC1-SW3-LEAF2][int:ETH5]#MLAG_KEEPALIVE#
   no switchport
   ip address 172.16.1.1/31
```

Собираем LAG который будет выступать PEER линком

```
interface Port-Channel1
   description [dev:DC1-SW3-LEAF2][int:PO1]#MLAG_PEER-LINK#
   switchport mode trunk
   switchport trunk group PEERLINK
   spanning-tree link-type point-to-point
!
interface Ethernet3
   description [dev:DC1-SW3-LEAF2][int:ETH3]#LAG_PO1#
   channel-group 1 mode active
!
interface Ethernet4
   description [dev:DC1-SW3-LEAF2][int:ETH4]#LAG_PO1#
   channel-group 1 mode active
```

Далее настраиваем параметры MLAG.

Указываем peer-link, keepalive и peer адреса соседа, локальный int для синхронизации MLAG

Действие при падении peer-link - отключение всех интерфейсов и recovery таймеры  - сначала поднимутся UPLINK`и (сойдутся протоколы маршрутизации), затем поднимутся линки в MLAG в сторону хостов

```
mlag configuration
   domain-id DC1-SW3-LEAF1-2
   local-interface Vlan4000
   peer-address 172.16.0.0
   peer-address heartbeat 172.16.1.0
   peer-link Port-Channel1
   dual-primary detection delay 1 action errdisable all-interfaces
   dual-primary recovery delay mlag 180 non-mlag 0
```

собираем MLAG в сторону конечного хоста

```
interface Port-Channel2
   description [dev:DC1-SW3-CE1][int:PO1]#UNI#
   switchport trunk allowed vlan 100,200
   switchport mode trunk
   mlag 1
!
interface Ethernet8
   description [dev:DC1-SW-CE1][int:ETH1]#MLAG_PO2#
   channel-group 2 mode active
!
```

### BGP

Настраиваем BGP по peer линку между MLAG коммутаторами и анонсируем общий Loopback его же указываем в качестве SRC интерфейса для VxLAN

```
interface Loopback1
   description #SHARED_VTEP#
   ip address 10.0.0.7/32
!
interface Vxlan1
   vxlan source-interface Loopback1
!
router bgp 64513
   neighbor 172.16.0.0 peer group PG_LEAF_MLAG
   neighbor 172.16.0.0 description [dev:DC1-SW3-LEAF2]#MLAG-PEER#
   !
   address-family ipv4
   neighbor PG_LEAF_MLAG activate
   neighbor PG_SPINE activate
   network 10.0.0.7/32
   !
```



## Multihoming

Настраиваем Po1 в сторону конечного хоста

указываем eth-segment для идентификации портов на разных коммутаторах в одном LAG интерфейсе

указываем в ручном режиме preference для выбора DF коммутатора

указываем RT для импорта 4 типа маршрутов между коммутаторами ESI сегменте

и system-id для LACP, что бы конечный хост воспринимал разные коммутаторы единой агрегацией 

```
interface Port-Channel1
   description [dev:DC1-SW3-CE2][int:PO1]#UNI#
   switchport trunk allowed vlan 100,200
   switchport mode trunk
   !
   evpn ethernet-segment
      identifier 0000:0000:0000:0000:1111
      designated-forwarder election algorithm preference 100
      route-target import 00:00:00:00:00:01
   lacp system-id 1234.1234.1234
!
interface Ethernet8
   description [dev:DC1-SW-CE2][int:ETH1]#ESI_PO1#
   channel-group 1 mode active
!
```

так же добавим механизм отслеживания состояния для предотвращения блекхолига трафика в случае падения UPLINK`ов

```
link tracking group LTG_NNI
   recovery delay 1
!
interface Ethernet1
   link tracking group LTG_NNI upstream
!
interface Ethernet2
   link tracking group LTG_NNI upstream 
!
interface Ethernet8
   link tracking group LTG_NNI downstream
!
```



## Customer

создаем для клиентов разные vlan

```
vlan 100,200
!
vrf instance 100
!
vrf instance 200
```

объединяем порты в LAG

```
interface Port-Channel1
   switchport trunk allowed vlan 100,200
   switchport mode trunk
!
interface Ethernet1
   channel-group 1 mode active
!
interface Ethernet2
   channel-group 1 mode active
```

назначаем ip адресацию клиентам и помещаем в разные vrf внутри коммутатора для изоляции (имитация двух разных хостов)

```
interface Vlan100
   vrf 100
   ip address 192.168.100.10/24
!
interface Vlan200
   vrf 200
   ip address 192.168.200.10/24
!
ip routing
ip routing vrf 100
ip routing vrf 200
!
ip route vrf 100 0.0.0.0/0 192.168.100.1
ip route vrf 200 0.0.0.0/0 192.168.200.1
```

# Проверка

## Связность между конечными хостами



## LACP



## Multihoming



### Type 1 routes



### Type 4 routes



## Проверка отказоустойчивости



## Дамп трафика