# Построение OVERLAY сети (VxLAN. EVPN L3)

## Цель работы 

Настроить маршрутизацию в рамках Overlay между клиентами

## План работы

- Настроить клиентов (VNI, маршрутизация между клиентами)
- Проверить связность между клиентами

## Тестовая среда

Схема сети в [PnetLab](../2.2%20-%20VxLAN.%20L3%20VNI/Attached%20files/LAB_2_2%20EVPN%20L3.unl)

![Схема сети](../2.2%20-%20VxLAN.%20L3%20VNI/Attached%20files/LAB_2-2.png)
Конфигурации сетевых элементов:

[DC1-SW3-LEAF1](../2.2%20-%20VxLAN.%20L3%20VNI/CFG/DC1-SW3-LEAF1.txt)

[DC1-SW3-LEAF2](../2.2%20-%20VxLAN.%20L3%20VNI/CFG/DC1-SW3-LEAF2.txt)

[DC1-SW3-LEAF3](../2.2%20-%20VxLAN.%20L3%20VNI/CFG/DC1-SW3-LEAF3.txt)

[DC1-SW3-SPINE1](../2.2%20-%20VxLAN.%20L3%20VNI/CFG/DC1-SW3-SPINE1.txt)

[DC1-SW3-SPINE2](../2.2%20-%20VxLAN.%20L3%20VNI/CFG/DC1-SW3-SPINE2.txt)


## Описание лабораторного стенда

Для клиента CST1 помимо l2 cвязности в VLAN 100,200 (сервисы 1,2) добавляется l3 связность (сервис 3) между его l2 доменами.
LEAF коммутаторы будут выступать в качестве шлюзов для конечных хостов клиента. 
Anycast GW - ip(mac) адрес шлюза будет единый для всех LEAF участвующих в сервисe. Для конечных хостов это будет значить, что при миграции на другой LEAF у них не изменятся атрибуты шлюза.

## Учет ID

VNI = порядковые значения
RD = loopback+порядковый id
RT = SPINE-AS+порядковый id

| VNI id | Service | Description |
| --- | --- | --- |
| 1 | [cst:1][svc:1] | vlan 100 |
| 2 | [cst:1][svc:2] | vlan 200 |
| 3 | [cst:2][svc:1] | vlan 300 |
| 4 | [cst:1][svc:3] | vrf CST1 |
| 5 | [cst:1][svc:2] | vlan 200 |

| RD | RT | Service | Description |
| --- | --- | --- | --- |
| 1 | 1 | [cst:1][svc:1][svc:2] | vlan-aware-bundle CST1 |
| 2 | 2 | [cst:2][svc:1] | vlan 300 |
| 3 | 3 | [cst:1][svc:3] | vrf CST1 |
| 4 | 4 | [cst:1][svc:2] | vlan 400 |



# Настройка клиентов (VNI, маршрутизация между клиентами)

> в качестве примера LEAF выступает DC1-SW3-LEAF1

```
vlan 100
   name [cvlan:100][cst:1][svc:1]
!
vlan 200
   name [cvlan:200][cst:1][svc:2]
!
vrf instance CST1
   description [cst:1][svc:3]
!
interface Vlan100
   vrf CST1
   ip address virtual 192.168.100.1/24
!
interface Vlan200
   vrf CST1
   ip address virtual 192.168.200.1/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 100 vni 20100
   vxlan vlan 200 vni 20200
   vxlan vlan 300 vni 20300
   vxlan vrf CST1 vni 30003
!
ip virtual-router mac-address 00:00:00:00:11:11
!
ip routing vrf CST1
!
router bgp 64513
   vlan-aware-bundle CST1
      rd 10.0.0.3:1
      route-target both 1:11
      redistribute learned
      vlan 100,200
   !   
   vrf CST1
      rd 10.0.0.3:1
      route-target import evpn 1:13
      route-target export evpn 1:13
      redistribute connected
!
```

аналогичным образом настроен DC1-SW3-LEAF3

---

# Проверка

проверку EVPN, interface VxLAN ... не осуществляем т.к. с прошлой работы ничего не поменялось. 

## Type 2 routes и таблицы маршрутизации


**Проверяем маршруты для L2 сервиса и маки на коммутаторах**

VLAN 100 (VNI 20100)

```
DC1-SW3-LEAF1#show bgp evpn vni 20100
BGP routing table information for VRF default
Router identifier 10.0.0.3, local AS number 64513
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.0.0.3:1 mac-ip 20100 0050.7966.6801
                                 -                     -       -       0       i
 * >      RD: 10.0.0.3:1 mac-ip 20100 0050.7966.6801 192.168.100.11
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.0.5:1 mac-ip 20100 0050.7966.6804
                                 10.0.0.5              -       100     0       64512 64515 i
 *  ec    RD: 10.0.0.5:1 mac-ip 20100 0050.7966.6804
                                 10.0.0.5              -       100     0       64512 64515 i
 * >Ec    RD: 10.0.0.5:1 mac-ip 20100 0050.7966.6804 192.168.100.4
                                 10.0.0.5              -       100     0       64512 64515 i
 *  ec    RD: 10.0.0.5:1 mac-ip 20100 0050.7966.6804 192.168.100.4
                                 10.0.0.5              -       100     0       64512 64515 i
 * >      RD: 10.0.0.3:1 imet 20100 10.0.0.3
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.0.5:1 imet 20100 10.0.0.5
                                 10.0.0.5              -       100     0       64512 64515 i
 *  ec    RD: 10.0.0.5:1 imet 20100 10.0.0.5
                                 10.0.0.5              -       100     0       64512 64515 i
```

Видим локальные mac и mac+ip маршруты, а так же mac и mac+ip маршруты хоста за DC1-SW3-LEAF3

```
DC1-SW3-LEAF1#show l2rib output vlan 100
0050.7966.6801, VLAN 100, seq 1, pref 16, learnedDynamicMac, source: Local Dynamic
   Ethernet8
0050.7966.6804, VLAN 100, seq 1, pref 16, evpnDynamicRemoteMac, source: BGP
   VTEP 10.0.0.5
```

MAC`и в VLAN на месте


```
DC1-SW3-LEAF1#show bgp evpn vni 20200
BGP routing table information for VRF default
Router identifier 10.0.0.3, local AS number 64513
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.0.0.5:1 mac-ip 20200 0050.7966.6803
                                 10.0.0.5              -       100     0       64512 64515 i
 *  ec    RD: 10.0.0.5:1 mac-ip 20200 0050.7966.6803
                                 10.0.0.5              -       100     0       64512 64515 i
 * >Ec    RD: 10.0.0.5:1 mac-ip 20200 0050.7966.6803 192.168.200.3
                                 10.0.0.5              -       100     0       64512 64515 i
 *  ec    RD: 10.0.0.5:1 mac-ip 20200 0050.7966.6803 192.168.200.3
                                 10.0.0.5              -       100     0       64512 64515 i
 * >      RD: 10.0.0.3:1 mac-ip 20200 0050.7966.6806
                                 -                     -       -       0       i
 * >      RD: 10.0.0.3:1 mac-ip 20200 0050.7966.6806 192.168.200.6
                                 -                     -       -       0       i
 * >      RD: 10.0.0.3:1 imet 20200 10.0.0.3
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.0.5:1 imet 20200 10.0.0.5
                                 10.0.0.5              -       100     0       64512 64515 i
 *  ec    RD: 10.0.0.5:1 imet 20200 10.0.0.5
                                 10.0.0.5              -       100     0       64512 64515 i
```

```
DC1-SW3-LEAF1#show l2rib output vlan 200
0050.7966.6803, VLAN 200, seq 1, pref 16, evpnDynamicRemoteMac, source: BGP
   VTEP 10.0.0.5
0050.7966.6806, VLAN 200, seq 1, pref 16, learnedDynamicMac, source: Local Dynamic
   Ethernet6
```

аналогичная картина для второго L2 сервиса VLAN 200 (VNI 20200)

**проверяем связность внутри l2 домена**

> DC1-SR-NODE1[VLAN100][192.168.100.11] <> DC1-SR-NODE4[VLAN100][192.168.100.4]

```
DC1-SR-NODE1> ping 192.168.100.4

84 bytes from 192.168.100.4 icmp_seq=1 ttl=64 time=19.577 ms
84 bytes from 192.168.100.4 icmp_seq=2 ttl=64 time=44.354 ms
84 bytes from 192.168.100.4 icmp_seq=3 ttl=64 time=22.728 ms
84 bytes from 192.168.100.4 icmp_seq=4 ttl=64 time=22.019 ms
84 bytes from 192.168.100.4 icmp_seq=5 ttl=64 time=18.924 ms
```

в дампе видно что ARP запрос не уходит дальше порта LEAF коммутатора (это означает что работает arp suppression, DC1-SW3-LEAF1 сам отвечает на ARP запрос т.к. у него есть данная информация о интересующем нас MAC адресе) - на ARISTA работает по умолчанию при включении SVI интерфейса для VLAN

![ARP1](../2.2%20-%20VxLAN.%20L3%20VNI/Attached%20files/ARP1.png)
![ARP2](../2.2%20-%20VxLAN.%20L3%20VNI/Attached%20files/ARP2.png)

ICMP пакеты передаются в VxLAN с VNI 20100, что говорит о том что работает коммутация через l2 VNI

![VxLAN VNI 100](../2.2%20-%20VxLAN.%20L3%20VNI/Attached%20files/VXLAN_VNI100.png)

**проверяем L3 связность между широковещательными доменами**

для начала проверим маршруты на DC1-SW3-LEAF1

```
DC1-SW3-LEAF1#show bgp evpn vni 30003
BGP routing table information for VRF default
Router identifier 10.0.0.3, local AS number 64513
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.0.0.3:1 mac-ip 20100 0050.7966.6801 192.168.100.11
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.0.5:1 mac-ip 20200 0050.7966.6803 192.168.200.3
                                 10.0.0.5              -       100     0       64512 64515 i
 *  ec    RD: 10.0.0.5:1 mac-ip 20200 0050.7966.6803 192.168.200.3
                                 10.0.0.5              -       100     0       64512 64515 i
 * >      RD: 10.0.0.3:1 ip-prefix 192.168.100.0/24
                                 -                     -       -       0       i
 * >      RD: 10.0.0.3:1 ip-prefix 192.168.200.0/24
                                 -                     -       -       0       i
```

```
DC1-SW3-LEAF1#show ip route vrf CST1

VRF: CST1
Codes: C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - Other BGP Routes,
       B I - iBGP, B E - eBGP, R - RIP, I L1 - IS-IS level 1,
       I L2 - IS-IS level 2, O3 - OSPFv3, A B - BGP Aggregate,
       A O - OSPF Summary, NG - Nexthop Group Static Route,
       V - VXLAN Control Service, M - Martian,
       DH - DHCP client installed default route,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

Gateway of last resort is not set

 C        192.168.100.0/24 is directly connected, Vlan100
 B E      192.168.200.3/32 [20/0] via VTEP 10.0.0.5 VNI 30003 router-mac 50:6d:26:f2:3c:e0 local-interface Vxlan1
 C        192.168.200.0/24 is directly connected, Vlan200
```
сеть 192.168.200.0/24 - коннектед, однако т.к. у нас имеется анонс хоста 192.168.200.3/32 по BGP, то исходящий трафик для хоста будет форвардится через VxLAN интерфейс с меткой VNI 30003

проверим пингом связность между хостами из разных VLAN

> DC1-SR-NODE1[VLAN100][192.168.100.11] <> DC1-SR-NODE3[VLAN200][192.168.200.3]

![VxLAN VNI 30003](../2.2%20-%20VxLAN.%20L3%20VNI/Attached%20files/VXLAN_VNI30003.png)

ICMP пакеты передаются в VxLAN с VNI 30003, в данном случае работает маршрутизация через l3 VNI