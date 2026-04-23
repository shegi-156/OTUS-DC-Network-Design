# Построение Underlay сети (OSPF)

**Цель:** настроить OSPF для UNDERLAY сети

[Схема сети]() в PnetLab

[Конфигурации]() сетевых элементов

# План работы

- Настроить OSPF на каждом сетевом элементе учитывая Best practices
- Убдиться в работоспособности протокола динамической маршрутизации (проверить соседство, убедиться в наличии маршрутов)
- Проверить связность до адресов полученных с помощью протокола динамической маршрутизации

## Настройка OSPF

> в качестве примера выступает DC1-SW3-SPINE1


В качестве RID использован ip интерфейса Lo0

Рефернсное значение для автоматического расчета cost выбрано значение 400Гб/с

Все интерфейсы в режиме passive, на интерфейсах с которых строится соседтсво passive выключается принудительно 

Максимальное значение балансировки по ecmp выбрано 4 (с учетом возможной модернизации и добавления новых SPINE)

```
!
router ospf 1
   router-id 10.0.0.1
   auto-cost reference-bandwidth 400000
   passive-interface default
   no passive-interface Ethernet1
   no passive-interface Ethernet2
   no passive-interface Ethernet3
   max-lsa 12000
   maximum-paths 4
!
```

При топологии с одним POD использовать BackBone area (area 0.0.0.0)

Тип OSPF интефейса p2p (т.к. все линки являются прямымы стыками)

MTU на интерфейсах настроен - 9000

Настроена аутентификация для установления соседства по OSPF

Так же настроено отслеживание соседства с помошью BFD

```
!
interface Ethernet3
   description DC1-SW3-LEAF3::Eth1
   mtu 9000
   no switchport
   ip address 10.0.1.4/31
   ip ospf neighbor bfd
   ip ospf network point-to-point
   ip ospf authentication message-digest
   ip ospf area 0.0.0.0
   ip ospf message-digest-key 1 md5 7 WBFBMU6tWMnnTGHupOoTGQ==
!
```

Для BFD глобально выбраны таймеры (Tx/Rx 100 ms x 3)

```
!
router bfd
   interval 100 min-rx 100 multiplier 3 default
!
```

## Проверка

После применения данных настроек на всех сетевых элементах проверяем состояние соседства OSPF

```
DC1-SW3-LEAF1#show ip ospf neighbor
Neighbor ID     Instance VRF      Pri State                  Dead Time   Address         Interface
10.0.0.1        1        default  0   FULL                   00:00:38    10.0.1.0        Ethernet1
10.0.0.2        1        default  0   FULL                   00:00:37    10.0.1.6        Ethernet2
```

```
DC1-SW3-LEAF2#show ip ospf neighbor
Neighbor ID     Instance VRF      Pri State                  Dead Time   Address         Interface
10.0.0.2        1        default  0   FULL                   00:00:29    10.0.1.8        Ethernet2
10.0.0.1        1        default  0   FULL                   00:00:33    10.0.1.2        Ethernet1
```

```
DC1-SW3-LEAF3#show ip ospf neighbor
Neighbor ID     Instance VRF      Pri State                  Dead Time   Address         Interface
10.0.0.1        1        default  0   FULL                   00:00:38    10.0.1.4        Ethernet1
10.0.0.2        1        default  0   FULL                   00:00:30    10.0.1.10       Ethernet2
```

```
DC1-SW3-SPINE1#show ip ospf neighbor
Neighbor ID     Instance VRF      Pri State                  Dead Time   Address         Interface
10.0.0.3        1        default  0   FULL                   00:00:30    10.0.1.1        Ethernet1
10.0.0.4        1        default  0   FULL                   00:00:37    10.0.1.3        Ethernet2
10.0.0.5        1        default  0   FULL                   00:00:36    10.0.1.5        Ethernet3
```

```
DC1-SW3-SPINE2#show ip ospf neighbor
Neighbor ID     Instance VRF      Pri State                  Dead Time   Address         Interface
10.0.0.4        1        default  0   FULL                   00:00:38    10.0.1.9        Ethernet2
10.0.0.3        1        default  0   FULL                   00:00:34    10.0.1.7        Ethernet1
10.0.0.5        1        default  0   FULL                   00:00:35    10.0.1.11       Ethernet3
```

после того как все соседсвта подняты и находятся в стосоянии FULL проверяем таблицу маршрутизации, что бы убедиться что все нужные сети аннонсируются 

```
DC1-SW3-LEAF1#show ip route ospf detail

VRF: default
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

 O        10.0.0.1/32 [110/410] via 10.0.1.0, Ethernet1 DC1-SW3-SPINE1::Eth1
 O        10.0.0.2/32 [110/410] via 10.0.1.6, Ethernet2 DC1-SW3-SPINE2::Eth1
 O        10.0.0.4/32 [110/810] via 10.0.1.0, Ethernet1 DC1-SW3-SPINE1::Eth1
                                via 10.0.1.6, Ethernet2 DC1-SW3-SPINE2::Eth1
 O        10.0.0.5/32 [110/810] via 10.0.1.0, Ethernet1 DC1-SW3-SPINE1::Eth1
                                via 10.0.1.6, Ethernet2 DC1-SW3-SPINE2::Eth1
 O        10.0.1.2/31 [110/800] via 10.0.1.0, Ethernet1 DC1-SW3-SPINE1::Eth1
 O        10.0.1.4/31 [110/800] via 10.0.1.0, Ethernet1 DC1-SW3-SPINE1::Eth1
 O        10.0.1.8/31 [110/800] via 10.0.1.6, Ethernet2 DC1-SW3-SPINE2::Eth1
 O        10.0.1.10/31 [110/800] via 10.0.1.6, Ethernet2 DC1-SW3-SPINE2::Eth1
```

```
DC1-SW3-LEAF2#show ip route ospf detail

VRF: default
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

 O        10.0.0.1/32 [110/410] via 10.0.1.2, Ethernet1 DC1-SW3-SPINE1::Eth2
 O        10.0.0.2/32 [110/410] via 10.0.1.8, Ethernet2 DC1-SW3-SPINE2::Eth2
 O        10.0.0.3/32 [110/810] via 10.0.1.2, Ethernet1 DC1-SW3-SPINE1::Eth2
                                via 10.0.1.8, Ethernet2 DC1-SW3-SPINE2::Eth2
 O        10.0.0.5/32 [110/810] via 10.0.1.2, Ethernet1 DC1-SW3-SPINE1::Eth2
                                via 10.0.1.8, Ethernet2 DC1-SW3-SPINE2::Eth2
 O        10.0.1.0/31 [110/800] via 10.0.1.2, Ethernet1 DC1-SW3-SPINE1::Eth2
 O        10.0.1.4/31 [110/800] via 10.0.1.2, Ethernet1 DC1-SW3-SPINE1::Eth2
 O        10.0.1.6/31 [110/800] via 10.0.1.8, Ethernet2 DC1-SW3-SPINE2::Eth2
 O        10.0.1.10/31 [110/800] via 10.0.1.8, Ethernet2 DC1-SW3-SPINE2::Eth2

```

```
DC1-SW3-LEAF3#show ip route ospf detail

VRF: default
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

 O        10.0.0.1/32 [110/410] via 10.0.1.4, Ethernet1 DC1-SW3-SPINE1::Eth3
 O        10.0.0.2/32 [110/410] via 10.0.1.10, Ethernet2 DC1-SW3-SPINE2::Eth3
 O        10.0.0.3/32 [110/810] via 10.0.1.4, Ethernet1 DC1-SW3-SPINE1::Eth3
                                via 10.0.1.10, Ethernet2 DC1-SW3-SPINE2::Eth3
 O        10.0.0.4/32 [110/810] via 10.0.1.4, Ethernet1 DC1-SW3-SPINE1::Eth3
                                via 10.0.1.10, Ethernet2 DC1-SW3-SPINE2::Eth3
 O        10.0.1.0/31 [110/800] via 10.0.1.4, Ethernet1 DC1-SW3-SPINE1::Eth3
 O        10.0.1.2/31 [110/800] via 10.0.1.4, Ethernet1 DC1-SW3-SPINE1::Eth3
 O        10.0.1.6/31 [110/800] via 10.0.1.10, Ethernet2 DC1-SW3-SPINE2::Eth3
 O        10.0.1.8/31 [110/800] via 10.0.1.10, Ethernet2 DC1-SW3-SPINE2::Eth3
```

Видим что Loopback'и Leaf'ов доступны через оба SPINE

```
DC1-SW3-SPINE1#show ip route ospf detail

VRF: default
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

 O        10.0.0.2/32 [110/810] via 10.0.1.1, Ethernet1 DC1-SW3-LEAF1::Eth1
                                via 10.0.1.3, Ethernet2 DC1-SW3-LEAF2::Eth1
                                via 10.0.1.5, Ethernet3 DC1-SW3-LEAF3::Eth1
 O        10.0.0.3/32 [110/410] via 10.0.1.1, Ethernet1 DC1-SW3-LEAF1::Eth1
 O        10.0.0.4/32 [110/410] via 10.0.1.3, Ethernet2 DC1-SW3-LEAF2::Eth1
 O        10.0.0.5/32 [110/410] via 10.0.1.5, Ethernet3 DC1-SW3-LEAF3::Eth1
 O        10.0.1.6/31 [110/800] via 10.0.1.1, Ethernet1 DC1-SW3-LEAF1::Eth1
 O        10.0.1.8/31 [110/800] via 10.0.1.3, Ethernet2 DC1-SW3-LEAF2::Eth1
 O        10.0.1.10/31 [110/800] via 10.0.1.5, Ethernet3 DC1-SW3-LEAF3::Eth1
```

```
DC1-SW3-SPINE2#show ip route ospf detail

VRF: default
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

 O        10.0.0.1/32 [110/810] via 10.0.1.7, Ethernet1 DC1-SW3-LEAF1::Eth2
                                via 10.0.1.9, Ethernet2 DC1-SW3-LEAF2::Eth2
                                via 10.0.1.11, Ethernet3 DC1-SW3-LEAF3::Eth2
 O        10.0.0.3/32 [110/410] via 10.0.1.7, Ethernet1 DC1-SW3-LEAF1::Eth2
 O        10.0.0.4/32 [110/410] via 10.0.1.9, Ethernet2 DC1-SW3-LEAF2::Eth2
 O        10.0.0.5/32 [110/410] via 10.0.1.11, Ethernet3 DC1-SW3-LEAF3::Eth2
 O        10.0.1.0/31 [110/800] via 10.0.1.7, Ethernet1 DC1-SW3-LEAF1::Eth2
 O        10.0.1.2/31 [110/800] via 10.0.1.9, Ethernet2 DC1-SW3-LEAF2::Eth2
 O        10.0.1.4/31 [110/800] via 10.0.1.11, Ethernet3 DC1-SW3-LEAF3::Eth2
```

между SPINE так же видим доступность через все 3 LEAF коммутатора

---

Далее проверяем L3 связность до между LO

*DC1-SW3-LEAF1 <> DC1-SW3-LEAF3*

```
DC1-SW3-LEAF1#ping 10.0.0.5 size 9000 df-bit source 10.0.0.3
PING 10.0.0.5 (10.0.0.5) from 10.0.0.3 : 8972(9000) bytes of data.
8980 bytes from 10.0.0.5: icmp_seq=1 ttl=63 time=10.3 ms
8980 bytes from 10.0.0.5: icmp_seq=2 ttl=63 time=12.9 ms
8980 bytes from 10.0.0.5: icmp_seq=3 ttl=63 time=14.3 ms
8980 bytes from 10.0.0.5: icmp_seq=4 ttl=63 time=17.4 ms
8980 bytes from 10.0.0.5: icmp_seq=5 ttl=63 time=10.7 ms

--- 10.0.0.5 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 51ms
rtt min/avg/max/mdev = 10.311/13.168/17.477/2.608 ms, pipe 2, ipg/ewma 12.821/11.758 ms

```

*DC1-SW3-LEAF3 <> DC1-SW3-LEAF2*

```
DC1-SW3-LEAF3#ping 10.0.0.4 size 9000 df-bit source 10.0.0.5
PING 10.0.0.4 (10.0.0.4) from 10.0.0.5 : 8972(9000) bytes of data.
8980 bytes from 10.0.0.4: icmp_seq=1 ttl=63 time=105 ms
8980 bytes from 10.0.0.4: icmp_seq=2 ttl=63 time=81.9 ms
8980 bytes from 10.0.0.4: icmp_seq=3 ttl=63 time=78.1 ms
8980 bytes from 10.0.0.4: icmp_seq=4 ttl=63 time=69.6 ms
8980 bytes from 10.0.0.4: icmp_seq=5 ttl=63 time=70.0 ms

--- 10.0.0.4 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 98ms
rtt min/avg/max/mdev = 69.670/81.099/105.725/13.191 ms, pipe 5, ipg/ewma 24.711/92.686 ms

```