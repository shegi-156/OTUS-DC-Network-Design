# Построение Underlay сети (IS-IS)

## Цель работы 

Настроить BGP для UNDERLAY сети

## План работы

- Настроить BGP на каждом сетевом элементе
- Убедиться в работоспособности протокола динамической маршрутизации (проверить соседство, убедиться в наличии маршрутов)
- Проверить связность до адресов полученных с помощью протокола динамической маршрутизации

## Тестовая среда

Схема сети в [PnetLab](../1.4%20-%20Building%20an%20Overlay%20Network%20(eBGP)/PnetLab/LAB_1_3.unl)
![Схема сети](../1.3%20-%20Building%20an%20Underlay%20Network%20(IS-IS)/Attached%20files/LAB_1_3.png)

Конфигурации сетевых элементов:

[DC1-SW3-LEAF1](../1.4%20-%20Building%20an%20Overlay%20Network%20(eBGP)/CFG/DC1-SW3-LEAF1.txt)

[DC1-SW3-LEAF2](../1.4%20-%20Building%20an%20Overlay%20Network%20(eBGP)/CFG/DC1-SW3-LEAF2.txt)

[DC1-SW3-LEAF3](../1.4%20-%20Building%20an%20Overlay%20Network%20(eBGP)/CFG/DC1-SW3-LEAF3.txt)

[DC1-SW3-SPINE1](../1.4%20-%20Building%20an%20Overlay%20Network%20(eBGP)/CFG/DC1-SW3-SPINE1.txt)

[DC1-SW3-SPINE2](../1.4%20-%20Building%20an%20Overlay%20Network%20(eBGP)/CFG/DC1-SW3-SPINE2.txt)



## Настройка BGP

При подготовке лабораторной работы были выбраны следующие подходы:

- ASN из приватного пула 64512 — 65534

- ASN распределялись по порядку, из принципа: все SPINE - одна AS, каждый LEAF - отдельная AS

В качестве RID - ip интерфейса Lo0

Максимальное значение балансировки по ecmp выбрано 4 (с учетом возможной модернизации и добавления новых SPINE)

Для удобства пиры объеденины в общую группу

BFD включен для всех BGP пиров в группе (глобально выбраны таймеры: Tx/Rx 100 ms x 3)

Включена аутентификация для BGP пиров в группе

> в качестве примера LEAF выступает DC1-SW3-LEAF1

```
router bfd
   interval 100 min-rx 100 multiplier 3 default
!
router bgp 64513
   router-id 10.0.0.3
   no bgp default ipv4-unicast
   timers bgp 3 9
   maximum-paths 4 ecmp 4
   neighbor PG_SPINE peer group
   neighbor PG_SPINE remote-as 64512
   neighbor PG_SPINE bfd
   neighbor PG_SPINE password 7 3ZMHuSYHdBchNfOxO2nKBg==
   neighbor PG_SPINE send-community standard extended large
   neighbor 10.0.1.0 peer group PG_SPINE
   neighbor 10.0.1.0 description DC1-SW3-SPINE1
   neighbor 10.0.1.6 peer group PG_SPINE
   neighbor 10.0.1.6 description DC1-SW3-SPINE2
   !
   address-family ipv4
      neighbor PG_SPINE activate
      network 10.0.0.3/32

```

> в качестве примера SPINE выступает DC1-SW3-SPINE1

Для удобства настройки и расширения фабрики пиры на SPINE`ах задаются диапазонами (bgp listen range)

```
peer-filter PF_LEAF
   10 match as-range 64513-65534 result accept
!
router bfd
   interval 100 min-rx 100 multiplier 3 default
!
router bgp 64512
   router-id 10.0.0.1
   no bgp default ipv4-unicast
   timers bgp 3 9
   maximum-paths 4 ecmp 4
   bgp listen range 10.0.1.0/24 peer-group PG_LEAF peer-filter PF_LEAF
   neighbor PG_LEAF peer group
   neighbor PG_LEAF bfd
   neighbor PG_LEAF password 7 z+yWsJee8ho4Ju8e6CN9sg==
   neighbor PG_LEAF send-community standard extended large
   !
   address-family ipv4
      neighbor PG_LEAF activate
      network 10.0.0.1/32
```





## Проверка

После применения данных настроек на всех сетевых элементах проверяем состояние BGP пиров и BFD

```
DC1-SW3-SPINE1#show ip bgp summary
BGP summary information for VRF default
Router identifier 10.0.0.1, local AS number 64512
Neighbor Status Codes: m - Under maintenance
  Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  10.0.1.1 4 64513           1498      1487    0    0 01:03:08 Estab   1      1
  10.0.1.3 4 64514           1485      1488    0    0 01:03:09 Estab   1      1
  10.0.1.5 4 64515           1486      1483    0    0 01:03:07 Estab   1      1

DC1-SW3-SPINE1#show bfd peers
VRF name: default
-----------------
DstAddr       MyDisc    YourDisc  Interface/Transport    Type           LastUp
--------- ----------- ----------- -------------------- ------- ----------------
10.0.1.1  2894087603  2145970137        Ethernet1(14)  normal   05/30/26 16:42
10.0.1.3   570844212   933826610        Ethernet2(15)  normal   05/30/26 16:42
10.0.1.5    68978836  3145243860        Ethernet3(16)  normal   05/30/26 16:42

   LastDown            LastDiag    State
-------------- ------------------- -----
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
```

```
DC1-SW3-SPINE2#show ip bgp summary
BGP summary information for VRF default
Router identifier 10.0.0.2, local AS number 64512
Neighbor Status Codes: m - Under maintenance
  Neighbor         V  AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  10.0.1.7         4  64513           1615      1375    0    0 01:08:18 Estab   1      1
  10.0.1.9         4  64514           1612      1376    0    0 01:08:18 Estab   1      1
  10.0.1.11        4  64515           1547      1321    0    0 01:05:44 Estab   1      1

DC1-SW3-SPINE2#show bfd peers
VRF name: default
-----------------
DstAddr        MyDisc    YourDisc  Interface/Transport    Type          LastUp
---------- ----------- ----------- -------------------- ------- ---------------
10.0.1.7    190373208  2942192196        Ethernet1(13)  normal  05/30/26 16:37
10.0.1.9   4294720614   276544599        Ethernet2(14)  normal  05/30/26 16:37
10.0.1.11  2809388190  3468187724        Ethernet3(15)  normal  05/30/26 16:39

   LastDown            LastDiag    State
-------------- ------------------- -----
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
```

```
DC1-SW3-LEAF1#show ip bgp summary
BGP summary information for VRF default
Router identifier 10.0.0.3, local AS number 64513
Neighbor Status Codes: m - Under maintenance
  Description              Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  DC1-SW3-SPINE1           10.0.1.0 4 64512           2040      2083    0    0 01:04:46 Estab   3      3
  DC1-SW3-SPINE2           10.0.1.6 4 64512           6293      7371    0    0 01:09:49 Estab   3      3

DC1-SW3-LEAF1#show bfd peers
VRF name: default
-----------------
DstAddr       MyDisc    YourDisc  Interface/Transport    Type           LastUp
--------- ----------- ----------- -------------------- ------- ----------------
10.0.1.0  2145970137  2894087603        Ethernet1(14)  normal   05/30/26 16:42
10.0.1.6  2942192196   190373208        Ethernet2(15)  normal   05/30/26 16:37

         LastDown            LastDiag    State
-------------------- ------------------- -----
   05/30/26 16:42       No Diagnostic       Up
   05/30/26 16:37       No Diagnostic       Up
```

```
DC1-SW3-LEAF2#show ip bgp summary
BGP summary information for VRF default
Router identifier 10.0.0.4, local AS number 64514
Neighbor Status Codes: m - Under maintenance
  Description              Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  DC1-SW3-SPINE1           10.0.1.2 4 64512           7577      7635    0    0 01:05:30 Estab   3      3
  DC1-SW3-SPINE2           10.0.1.8 4 64512           6272      7303    0    0 01:10:31 Estab   3      3

DC1-SW3-LEAF2#show bfd peers
VRF name: default
-----------------
DstAddr      MyDisc    YourDisc  Interface/Transport     Type           LastUp
--------- ---------- ----------- -------------------- -------- ----------------
10.0.1.2  933826610   570844212        Ethernet1(14)   normal   05/30/26 16:42
10.0.1.8  276544599  4294720614        Ethernet2(15)   normal   05/30/26 16:37

         LastDown            LastDiag    State
-------------------- ------------------- -----
   05/30/26 16:42       No Diagnostic       Up
   05/30/26 16:37       No Diagnostic       Up
```

```
DC1-SW3-LEAF3#show ip bgp summary
BGP summary information for VRF default
Router identifier 10.0.0.5, local AS number 64515
Neighbor Status Codes: m - Under maintenance
  Description              Neighbor  V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  DC1-SW3-SPINE1           10.0.1.4  4 64512           7606      7671    0    0 01:06:07 Estab   3      3
  DC1-SW3-SPINE2           10.0.1.10 4 64512           6261      7314    0    0 01:08:36 Estab   3      3

DC1-SW3-LEAF3#show bfd peers
VRF name: default
-----------------
DstAddr        MyDisc    YourDisc  Interface/Transport    Type          LastUp
---------- ----------- ----------- -------------------- ------- ---------------
10.0.1.4   3145243860    68978836        Ethernet1(14)  normal  05/30/26 16:42
10.0.1.10  3468187724  2809388190        Ethernet2(15)  normal  05/30/26 16:39

         LastDown            LastDiag    State
-------------------- ------------------- -----
   05/30/26 16:42       No Diagnostic       Up
   05/30/26 16:39       No Diagnostic       Up
```

Все соседства находятся в состоянии established, проверяем таблицу BGP (для компактности примет только с DC1-SW3-LEAF3)

```
DC1-SW3-LEAF3#show ip bgp
BGP routing table information for VRF default
Router identifier 10.0.0.5, local AS number 64515
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.0.0.1/32            10.0.1.4              0       -          100     0       64512 i
 * >      10.0.0.2/32            10.0.1.10             0       -          100     0       64512 i
 * >Ec    10.0.0.3/32            10.0.1.10             0       -          100     0       64512 64513 i
 *  ec    10.0.0.3/32            10.0.1.4              0       -          100     0       64512 64513 i
 * >Ec    10.0.0.4/32            10.0.1.10             0       -          100     0       64512 64514 i
 *  ec    10.0.0.4/32            10.0.1.4              0       -          100     0       64512 64514 i
 * >      10.0.0.5/32            -                     -       -          -       0       i
```

В таблице BGP вся необходимая информация о маршрутах к лупбекам присутствует, проверяем таблицу маршрутизации протокола BGP

```
DC1-SW3-LEAF3#show ip route bgp detail

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

 B E      10.0.0.1/32 [200/0] via 10.0.1.4, Ethernet1 DC1-SW3-SPINE1::Eth3
 B E      10.0.0.2/32 [200/0] via 10.0.1.10, Ethernet2 DC1-SW3-SPINE2::Eth3
 B E      10.0.0.3/32 [200/0] via 10.0.1.4, Ethernet1 DC1-SW3-SPINE1::Eth3
                              via 10.0.1.10, Ethernet2 DC1-SW3-SPINE2::Eth3
 B E      10.0.0.4/32 [200/0] via 10.0.1.4, Ethernet1 DC1-SW3-SPINE1::Eth3
                              via 10.0.1.10, Ethernet2 DC1-SW3-SPINE2::Eth3
```

```
DC1-SW3-LEAF1#show ip route bgp detail

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

 B E      10.0.0.1/32 [200/0] via 10.0.1.0, Ethernet1 DC1-SW3-SPINE1::Eth1
 B E      10.0.0.2/32 [200/0] via 10.0.1.6, Ethernet2 DC1-SW3-SPINE2::Eth1
 B E      10.0.0.4/32 [200/0] via 10.0.1.0, Ethernet1 DC1-SW3-SPINE1::Eth1
                              via 10.0.1.6, Ethernet2 DC1-SW3-SPINE2::Eth1
 B E      10.0.0.5/32 [200/0] via 10.0.1.0, Ethernet1 DC1-SW3-SPINE1::Eth1
                              via 10.0.1.6, Ethernet2 DC1-SW3-SPINE2::Eth1
```

```
DC1-SW3-LEAF2#show ip route bgp detail

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

 B E      10.0.0.1/32 [200/0] via 10.0.1.2, Ethernet1 DC1-SW3-SPINE1::Eth2
 B E      10.0.0.2/32 [200/0] via 10.0.1.8, Ethernet2 DC1-SW3-SPINE2::Eth2
 B E      10.0.0.3/32 [200/0] via 10.0.1.2, Ethernet1 DC1-SW3-SPINE1::Eth2
                              via 10.0.1.8, Ethernet2 DC1-SW3-SPINE2::Eth2
 B E      10.0.0.5/32 [200/0] via 10.0.1.2, Ethernet1 DC1-SW3-SPINE1::Eth2
                              via 10.0.1.8, Ethernet2 DC1-SW3-SPINE2::Eth2
```

```
DC1-SW3-SPINE1#show ip route bgp detail

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

 B E      10.0.0.3/32 [200/0] via 10.0.1.1, Ethernet1 DC1-SW3-LEAF1::Eth1
 B E      10.0.0.4/32 [200/0] via 10.0.1.3, Ethernet2 DC1-SW3-LEAF2::Eth1
 B E      10.0.0.5/32 [200/0] via 10.0.1.5, Ethernet3 DC1-SW3-LEAF3::Eth1
```

```
DC1-SW3-SPINE2#show ip route bgp detail

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

 B E      10.0.0.3/32 [200/0] via 10.0.1.7, Ethernet1 DC1-SW3-LEAF1::Eth2
 B E      10.0.0.4/32 [200/0] via 10.0.1.9, Ethernet2 DC1-SW3-LEAF2::Eth2
 B E      10.0.0.5/32 [200/0] via 10.0.1.11, Ethernet3 DC1-SW3-LEAF3::Eth2

```

Доступность адресов lo через равноценные линки имеется

---

Далее проверяем L3 связность до между LO

*DC1-SW3-LEAF1 <> DC1-SW3-LEAF3*

```
DC1-SW3-LEAF1#ping 10.0.0.5 size 9000 df-bit source lo0
PING 10.0.0.5 (10.0.0.5) from 10.0.0.3 : 8972(9000) bytes of data.
8980 bytes from 10.0.0.5: icmp_seq=1 ttl=63 time=22.8 ms
8980 bytes from 10.0.0.5: icmp_seq=2 ttl=63 time=18.0 ms
8980 bytes from 10.0.0.5: icmp_seq=3 ttl=63 time=15.1 ms
8980 bytes from 10.0.0.5: icmp_seq=4 ttl=63 time=9.36 ms
8980 bytes from 10.0.0.5: icmp_seq=5 ttl=63 time=10.6 ms

--- 10.0.0.5 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 77ms
rtt min/avg/max/mdev = 9.368/15.213/22.845/4.914 ms, pipe 2, ipg/ewma 19.452/18.710 ms
```

*DC1-SW3-LEAF3 <> DC1-SW3-LEAF2*

```
DC1-SW3-LEAF3#ping 10.0.0.4 size 9000 df-bit source lo0
PING 10.0.0.4 (10.0.0.4) from 10.0.0.5 : 8972(9000) bytes of data.
8980 bytes from 10.0.0.4: icmp_seq=1 ttl=63 time=16.4 ms
8980 bytes from 10.0.0.4: icmp_seq=2 ttl=63 time=12.5 ms
8980 bytes from 10.0.0.4: icmp_seq=3 ttl=63 time=15.2 ms
8980 bytes from 10.0.0.4: icmp_seq=4 ttl=63 time=10.3 ms
8980 bytes from 10.0.0.4: icmp_seq=5 ttl=63 time=10.6 ms

--- 10.0.0.4 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 71ms
rtt min/avg/max/mdev = 10.346/13.048/16.496/2.437 ms, ipg/ewma 17.812/14.640 ms
```