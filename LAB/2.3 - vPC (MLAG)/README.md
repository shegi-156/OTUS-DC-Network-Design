# VXLAN. Multihoming

## Цель работы 

Настроить отказоустойчивое подключение клиентов с использованием EVPN Multihoming.



## План работы

- Подключить клиентов 2-я линками к различным Leaf
- Настроить агрегированный канал со стороны клиента
- Настроить multihoming для работы в Overlay сети. Если используете Cisco NXOS - vPC, если иной вендор - то ESI LAG (либо MC-LAG с поддержкой VXLAN)



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

**ASN:64513** (DC1-SW3-LEAF1,2) - MLAG 

**ASN:64514** (DC1-SW3-LEAF3,4) - EVPN multihoming

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

Создадим отдельный технологический VRF

```
vrf instance MLAG
   description [tech:MLAG]#KEEPALIVE#
!
ip routing vrf MLAG
```

Поместим в VRF интерфейсы для peer-to-peer и keepalve связности.

для KEEPALIVE - отдельный физический линк между LEAF

для PEER-LINK - SVI VLAN 4000 выключим для него STP

```
vlan 4000
   name [cvlan:4000][tech:MLAG]#PEERVLAN
!
no spanning-tree vlan-id 4000
!
interface Vlan4000
   description [cvlan:4000][tech:MLAG]#MLAG_PEER-ADDRESS#
   vrf MLAG
   ip address 172.16.0.1/31
!
interface Ethernet5
   description [dev:DC1-SW3-LEAF2][int:ETH5]#MLAG_KEEPALIVE#
   no switchport
   vrf MLAG
   ip address 172.16.1.1/31
```

Собираем LAG интерфейс который будет выступать PEER линком

```
interface Port-Channel1
   description [dev:DC1-SW3-LEAF2][int:PO1]#MLAG_PEER-LINK#
   switchport trunk allowed vlan 4000
   switchport mode trunk
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

Далее настраиваем MLAG.

Указываем peer-link, keepalive и peer адреса соседа, локальный int для синхронизации MLAG

Действие при падении peer-link - отключение всех интерфейсов и recovery таймеры  - сначала поднимутся UPLINK`и (сойдутся протоколы маршрутизации), затем поднимутся линки в MLAG в сторону хостов

```
mlag configuration
   domain-id DC1-SW3-LEAF1-2
   local-interface Vlan4000
   peer-address 172.16.0.0
   peer-address heartbeat 172.16.1.0 vrf MLAG
   peer-link Port-Channel1
   dual-primary detection delay 1 action errdisable all-interfaces
   dual-primary recovery delay mlag 60 non-mlag 0
   reload-delay mlag 60
```



### 




## Multihoming



## Customer



# Проверка

## Связность между конечными хостами в разных VLAN




## Type 1 routes



## Type 4 routes



## Дамп трафика