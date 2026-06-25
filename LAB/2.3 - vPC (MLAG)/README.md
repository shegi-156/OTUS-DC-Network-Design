# VXLAN. Multihoming

## Цель работы 

Настроить отказоустойчивое подключение клиентов с использованием EVPN Multihoming.



## План работы

- Подключить клиентов 2-я линками к различным Leaf
- Настроить агрегированный канал со стороны клиента
- Настроить multihoming для работы в Overlay сети.



## Тестовая среда

Схема сети в [PnetLab](../2.3%20-%20vPC%20(MLAG)/Attached%20files/LAB_2_3%20EVPN%20VPC.unl)

Конфигурации сетевых элементов

**LEAF:**

[DC1-SW3-LEAF1](../2.3%20-%20vPC%20(MLAG)/CFG/DC1-SW3-LEAF1.txt)

[DC1-SW3-LEAF2](../2.3%20-%20vPC%20(MLAG)/CFG/DC1-SW3-LEAF2.txt)

[DC1-SW3-LEAF3](../2.3%20-%20vPC%20(MLAG)/CFG/DC1-SW3-LEAF3.txt)

[DC1-SW3-LEAF4](../2.3%20-%20vPC%20(MLAG)/CFG/DC1-SW3-LEAF4.txt)

**SPINE:**

[DC1-SW3-SPINE1](../2.3%20-%20vPC%20(MLAG)/CFG/DC1-SW3-SPINE1.txt)

[DC1-SW3-SPINE2](../2.3%20-%20vPC%20(MLAG)/CFG/DC1-SW3-SPINE2.txt)

**CE:**

[DC1-SW3-CE1](../2.3%20-%20vPC%20(MLAG)/CFG/DC1-SW3-CE1.txt)

[DC1-SW3-CE2](../2.3%20-%20vPC%20(MLAG)/CFG/DC1-SW3-CE2.txt)



## Описание лабораторного стенда

![alt text](../2.3%20-%20vPC%20(MLAG)/Attached%20files/LAB_2-3.png)

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

## LACP

```
DC1-SW3-LEAF1#show mlag
MLAG Configuration:
domain-id                          :     DC1-SW3-LEAF1-2
local-interface                    :            Vlan4000
peer-address                       :          172.16.0.0
peer-link                          :       Port-Channel1
hb-peer-address                    :          172.16.1.0
peer-config                        :          consistent

MLAG Status:
state                              :              Active
negotiation status                 :           Connected
peer-link status                   :                  Up
local-int status                   :                  Up
system-id                          :   52:70:54:d9:59:47
dual-primary detection             :          Configured
dual-primary interface errdisabled :               False

MLAG Ports:
Disabled                           :                   0
Configured                         :                   0
Inactive                           :                   0
Active-partial                     :                   0
Active-full                        :                   1
```

```
DC1-SW3-LEAF1#show l2Rib output detail
0000.0000.1111, VLAN 4000, seq 1, pref 64, peerStaticMac, source: MLAG   Port-Channel1
5070.54d9.5947, VLAN 200, seq 1, pref 64, peerStaticMac, source: MLAG   Port-Channel1
5070.54d9.5947, VLAN 4000, seq 1, pref 64, peerStaticMac, source: MLAG   Port-Channel1
5070.54d9.5947, VLAN 100, seq 1, pref 64, peerStaticMac, source: MLAG   Port-Channel1
5071.e982.5513, VLAN 100, seq 1, pref 16, evpnDynamicRemoteMac, source: BGP
   Load Balance entry 15: 2-way
      VTEP 10.0.0.5
      VTEP 10.0.0.6
5070.54d9.5947, VLAN 4094, seq 1, pref 64, peerStaticMac, source: MLAG   Port-Channel1
0000.0000.1111, VLAN 4094, seq 1, pref 64, peerStaticMac, source: MLAG   Port-Channel1
5071.e982.5513, VLAN 200, seq 1, pref 16, evpnDynamicRemoteMac, source: BGP
   Load Balance entry 15: 2-way
      VTEP 10.0.0.5
      VTEP 10.0.0.6
50a8.ec8a.deef, VLAN 200, seq 1, pref 16, learnedDynamicMac, source: Local Dynamic   Port-Channel2
50a8.ec8a.deef, VLAN 100, seq 1, pref 16, learnedDynamicMac, source: Local Dynamic   Port-Channel2
```

```
DC1-SW3-LEAF2#show ip route vrf CST1
 B E      192.168.100.20/32 [20/0] via VTEP 10.0.0.5 VNI 3 router-mac 50:6d:26:f2:3c:e0 local-interface Vxlan1
                                   via VTEP 10.0.0.6 VNI 3 router-mac 50:63:37:d5:4e:05 local-interface Vxlan1
 C        192.168.100.0/24 is directly connected, Vlan100
 B E      192.168.200.20/32 [20/0] via VTEP 10.0.0.5 VNI 3 router-mac 50:6d:26:f2:3c:e0 local-interface Vxlan1
                                   via VTEP 10.0.0.6 VNI 3 router-mac 50:63:37:d5:4e:05 local-interface Vxlan1
 C        192.168.200.0/24 is directly connected, Vlan200
```

## Multihoming

```
DC1-SW3-LEAF3#show bgp evpn instance
EVPN instance: VLAN-aware bundle CST1
  Route distinguisher: 10.0.0.5:1
  Route target import: Route-Target-AS:64512:1
  Route target export: Route-Target-AS:64512:1
  Service interface: VLAN-aware bundle
  Local VXLAN IP address: 10.0.0.5
  VXLAN: enabled
  MPLS: disabled
  Local ethernet segment:
    ESI: 0000:0000:0000:0000:1111
      Interface: Port-Channel1
      Mode: all-active
      State: up
      ES-Import RT: 00:00:00:00:00:01
      DF election algorithm: preference
      Designated forwarder: 10.0.0.5
      Non-Designated forwarder: 10.0.0.6
```

```
DC1-SW3-LEAF4#show bgp evpn instance
EVPN instance: VLAN-aware bundle CST1
  Route distinguisher: 10.0.0.6:1
  Route target import: Route-Target-AS:64512:1
  Route target export: Route-Target-AS:64512:1
  Service interface: VLAN-aware bundle
  Local VXLAN IP address: 10.0.0.6
  VXLAN: enabled
  MPLS: disabled
  Local ethernet segment:
    ESI: 0000:0000:0000:0000:1111
      Interface: Port-Channel1
      Mode: all-active
      State: up
      ES-Import RT: 00:00:00:00:00:01
      DF election algorithm: preference
      Designated forwarder: 10.0.0.5
      Non-Designated forwarder: 10.0.0.6
```


### Type 1 routes

```
DC1-SW3-LEAF2#show bgp evpn route-type auto-discovery
BGP routing table information for VRF default
Router identifier 10.0.0.4, local AS number 64513
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.0.0.5:1 auto-discovery 1 0000:0000:0000:0000:1111
                                 10.0.0.5              -       100     0       64512 64514 i
 *  ec    RD: 10.0.0.5:1 auto-discovery 1 0000:0000:0000:0000:1111
                                 10.0.0.5              -       100     0       64512 64514 i
 * >Ec    RD: 10.0.0.6:1 auto-discovery 1 0000:0000:0000:0000:1111
                                 10.0.0.6              -       100     0       64512 64515 i
 *  ec    RD: 10.0.0.6:1 auto-discovery 1 0000:0000:0000:0000:1111
                                 10.0.0.6              -       100     0       64512 64515 i
 * >Ec    RD: 10.0.0.5:1 auto-discovery 2 0000:0000:0000:0000:1111
                                 10.0.0.5              -       100     0       64512 64514 i
 *  ec    RD: 10.0.0.5:1 auto-discovery 2 0000:0000:0000:0000:1111
                                 10.0.0.5              -       100     0       64512 64514 i
 * >Ec    RD: 10.0.0.6:1 auto-discovery 2 0000:0000:0000:0000:1111
                                 10.0.0.6              -       100     0       64512 64515 i
 *  ec    RD: 10.0.0.6:1 auto-discovery 2 0000:0000:0000:0000:1111
                                 10.0.0.6              -       100     0       64512 64515 i
 * >Ec    RD: 10.0.0.5:1 auto-discovery 0000:0000:0000:0000:1111
                                 10.0.0.5              -       100     0       64512 64514 i
 *  ec    RD: 10.0.0.5:1 auto-discovery 0000:0000:0000:0000:1111
                                 10.0.0.5              -       100     0       64512 64514 i
 * >Ec    RD: 10.0.0.6:1 auto-discovery 0000:0000:0000:0000:1111
                                 10.0.0.6              -       100     0       64512 64515 i
 *  ec    RD: 10.0.0.6:1 auto-discovery 0000:0000:0000:0000:1111
                                 10.0.0.6              -       100     0       64512 64515 i
```

### Type 4 routes

```
DC1-SW3-LEAF3#show bgp evpn route-type ethernet-segment
BGP routing table information for VRF default
Router identifier 10.0.0.5, local AS number 64514
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.0.0.5:1 ethernet-segment 0000:0000:0000:0000:1111 10.0.0.5
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.0.6:1 ethernet-segment 0000:0000:0000:0000:1111 10.0.0.6
                                 10.0.0.6              -       100     0       64512 64515 i
 *  ec    RD: 10.0.0.6:1 ethernet-segment 0000:0000:0000:0000:1111 10.0.0.6
                                 10.0.0.6              -       100     0       64512 64515 i
```

```
DC1-SW3-LEAF4#show bgp evpn route-type ethernet-segment
BGP routing table information for VRF default
Router identifier 10.0.0.6, local AS number 64515
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.0.0.5:1 ethernet-segment 0000:0000:0000:0000:1111 10.0.0.5
                                 10.0.0.5              -       100     0       64512 64514 i
 *  ec    RD: 10.0.0.5:1 ethernet-segment 0000:0000:0000:0000:1111 10.0.0.5
                                 10.0.0.5              -       100     0       64512 64514 i
 * >      RD: 10.0.0.6:1 ethernet-segment 0000:0000:0000:0000:1111 10.0.0.6
                                 -                     -       -       0       i
```

## Связность между конечными хостами

VLAN 100 (L2 VNI - 1) **DC1-SW3-CE1 <> DC1-SW3-CE2** 

```
DC1-SW3-CE1#ping vrf 100 192.168.100.20 source 192.168.100.10
PING 192.168.100.20 (192.168.100.20) from 192.168.100.10 : 72(100) bytes of data.
80 bytes from 192.168.100.20: icmp_seq=1 ttl=64 time=30.2 ms
80 bytes from 192.168.100.20: icmp_seq=2 ttl=64 time=27.8 ms
80 bytes from 192.168.100.20: icmp_seq=3 ttl=64 time=27.7 ms
80 bytes from 192.168.100.20: icmp_seq=4 ttl=64 time=26.9 ms
80 bytes from 192.168.100.20: icmp_seq=5 ttl=64 time=28.5 ms
```

VLAN 200 (L2 VNI - 2) **DC1-SW3-CE1 <> DC1-SW3-CE2**

```
DC1-SW3-CE1#ping vrf 200 192.168.200.20 source 192.168.200.10
PING 192.168.200.20 (192.168.200.20) from 192.168.200.10 : 72(100) bytes of data.
80 bytes from 192.168.200.20: icmp_seq=1 ttl=64 time=224 ms
80 bytes from 192.168.200.20: icmp_seq=2 ttl=64 time=221 ms
80 bytes from 192.168.200.20: icmp_seq=3 ttl=64 time=316 ms
80 bytes from 192.168.200.20: icmp_seq=4 ttl=64 time=312 ms
80 bytes from 192.168.200.20: icmp_seq=5 ttl=64 time=396 ms
```

VRF CST1 (L3 VNI - 3) **DC1-SW3-CE1 <> DC1-SW3-CE2**

```
DC1-SW3-CE1#ping vrf 100 192.168.200.20 source 192.168.100.10
PING 192.168.200.20 (192.168.200.20) from 192.168.100.10 : 72(100) bytes of data.
80 bytes from 192.168.200.20: icmp_seq=1 ttl=62 time=76.0 ms
80 bytes from 192.168.200.20: icmp_seq=2 ttl=62 time=77.4 ms
80 bytes from 192.168.200.20: icmp_seq=3 ttl=62 time=73.3 ms
80 bytes from 192.168.200.20: icmp_seq=4 ttl=62 time=70.0 ms
80 bytes from 192.168.200.20: icmp_seq=5 ttl=62 time=66.9 ms
```

![VRF CST1](../2.3%20-%20vPC%20(MLAG)/Attached%20files/Ping_VRF_CST1.png)

## Проверка отказоустойчивости

при падении ETH1 DC1-SW3-CE2 видим что ICMP реквесты стали проходить через ETH2

![CE2 Down ETH1](../2.3%20-%20vPC%20(MLAG)/Attached%20files/down_port_CE2.png)

при падении ETH2 DC1-SW3-CE1 видим что реквесты стали проходить через ETH1

![CE1 Down ETH2](../2.3%20-%20vPC%20(MLAG)/Attached%20files/down_port_CE1.png)