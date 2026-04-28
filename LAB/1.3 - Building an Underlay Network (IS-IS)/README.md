# Построение Underlay сети (IS-IS)

## Цель работы 

Настроить OSPF для UNDERLAY сети

## План работы

- Настроить IS-IS на каждом сетевом элементе учитывая Best practices
- Убедиться в работоспособности протокола динамической маршрутизации (проверить соседство, убедиться в наличии маршрутов)
- Проверить связность до адресов полученных с помощью протокола динамической маршрутизации

## Тестовая среда

Схема сети в [PnetLab](..//1.3%20-%20Building%20an%20Underlay%20Network%20(IS-IS)/PnetLab/LAB_1_3.unl)
![Схема сети](../1.3%20-%20Building%20an%20Underlay%20Network%20(IS-IS)/Attached%20files/LAB_1_3.png)

Конфигурации сетевых элементов:

[DC1-SW3-LEAF1](../1.3%20-%20Building%20an%20Underlay%20Network%20(IS-IS)/CFG/DC1-SW3-LEAF1.txt)

[DC1-SW3-LEAF2](../1.3%20-%20Building%20an%20Underlay%20Network%20(IS-IS)/CFG/DC1-SW3-LEAF2.txt)

[DC1-SW3-LEAF3](../1.3%20-%20Building%20an%20Underlay%20Network%20(IS-IS)/CFG/DC1-SW3-LEAF3.txt)

[DC1-SW3-SPINE1](../1.3%20-%20Building%20an%20Underlay%20Network%20(IS-IS)/CFG/DC1-SW3-SPINE1.txt)

[DC1-SW3-SPINE2](../1.3%20-%20Building%20an%20Underlay%20Network%20(IS-IS)/CFG/DC1-SW3-SPINE2.txt)



## Настройка IS-IS

> в качестве примера выступает DC1-SW3-SPINE1

NET
- AFI = 49 (local)
- Area ID = 1 (DC1)
- System ID = loopback ip
- Selector = 00 (сам хост)

В качестве RID - ip интерфейса Lo0

IS-Hostname - указан для наглядности при диагностике

При топологии с одним POD IS-Level выбран L2-only (backbone)

Установлен overload бит при включении сетевого элемента (коммутатор не будет использоваться для пересылки трафика, до сходимости BGP/таймаута)

Референсное значение для автоматического расчета метрики IGP протокола выбрано значение 400 GBps

Интерфейс lo0 в режиме passive (на интерфейсах с которых строится соседство не включается)

Максимальное значение балансировки по ecmp выбрано 4 (с учетом возможной модернизации и добавления новых SPINE)

BFD включен для всех интерфейсов участвующих в процессе IS-IS (глобально выбраны таймеры: Tx/Rx 100 ms x 3)

Включена аутентификация для процесса IS-IS (применяется к LSP, CSNP и PSNP)

```
router bfd
   interval 100 min-rx 100 multiplier 3 default
!
router isis 1
   net 49.0001.0100.0000.0001.00
   is-hostname DC1-SW3-SPINE1
   router-id ipv4 10.0.0.1
   is-type level-2
   authentication mode md5
   authentication key 7 2FxrkNYPBNNpn3hz5KblEA==
   !
   address-family ipv4 unicast
      maximum-paths 4
      bfd all-interfaces
   !
   metric profile MP_REF_BW_400
      metric ratio 400 gbps
```

Тип IS-IS интерфейса p2p (т.к. все линки являются прямыми стыками)

MTU на интерфейсах настроен - 9000

Настроена аутентификация для установления соседства IS-IS

```
!
interface Ethernet1
   description DC1-SW3-LEAF1::Eth1
   mtu 9000
   no switchport
   ip address 10.0.1.0/31
   isis enable 1
   isis metric profile MP_REF_BW_400
   isis network point-to-point
   isis authentication mode md5
   isis authentication key 7 2FxrkNYPBNNpn3hz5KblEA==
!
```


## Проверка

После применения данных настроек на всех сетевых элементах проверяем состояние соседства IS-IS и BFD

```
DC1-SW3-SPINE1#show isis neighbors

Instance  VRF      System Id        Type Interface          SNPA              State Hold time   Circuit Id
1         default  DC1-SW3-LEAF1    L2   Ethernet1          P2P               UP    29          0D
1         default  DC1-SW3-LEAF2    L2   Ethernet2          P2P               UP    23          0D
1         default  DC1-SW3-LEAF3    L2   Ethernet3          P2P               UP    23          0D

DC1-SW3-SPINE1#show bfd peers
VRF name: default
-----------------
DstAddr       MyDisc    YourDisc  Interface/Transport    Type           LastUp
--------- ----------- ----------- -------------------- ------- ----------------
10.0.1.1  1241019198  3670280619        Ethernet1(13)  normal   04/28/26 19:38
10.0.1.3   922459779  3628520478        Ethernet2(14)  normal   04/28/26 19:22
10.0.1.5  1905081756  1028976166        Ethernet3(15)  normal   04/28/26 17:46

   LastDown            LastDiag    State
-------------- ------------------- -----
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up

```

```
DC1-SW3-SPINE2#show isis neighbors

Instance  VRF      System Id        Type Interface          SNPA              State Hold time   Circuit Id
1         default  DC1-SW3-LEAF1    L2   Ethernet1          P2P               UP    29          0E
1         default  DC1-SW3-LEAF2    L2   Ethernet2          P2P               UP    27          0E
1         default  DC1-SW3-LEAF3    L2   Ethernet3          P2P               UP    24          0E

DC1-SW3-SPINE2#show bfd peers
VRF name: default
-----------------
DstAddr        MyDisc    YourDisc  Interface/Transport    Type          LastUp
---------- ----------- ----------- -------------------- ------- ---------------
10.0.1.7    402626174  2589642531        Ethernet1(13)  normal  04/28/26 20:23
10.0.1.9   4239833456  3070056366        Ethernet2(14)  normal  04/28/26 20:23
10.0.1.11   451113844  3602833428        Ethernet3(15)  normal  04/28/26 20:23

   LastDown            LastDiag    State
-------------- ------------------- -----
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up

```

```
DC1-SW3-LEAF1#show isis neighbors

Instance  VRF      System Id        Type Interface          SNPA              State Hold time   Circuit Id
1         default  DC1-SW3-SPINE1   L2   Ethernet1          P2P               UP    22          0D
1         default  DC1-SW3-SPINE2   L2   Ethernet2          P2P               UP    23          0D

DC1-SW3-LEAF1#show bfd peers
VRF name: default
-----------------
DstAddr       MyDisc    YourDisc  Interface/Transport    Type           LastUp
--------- ----------- ----------- -------------------- ------- ----------------
10.0.1.0  3670280619  1241019198        Ethernet1(13)  normal   04/28/26 19:38
10.0.1.6  2589642531   402626174        Ethernet2(14)  normal   04/28/26 20:23

   LastDown            LastDiag    State
-------------- ------------------- -----
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up

```

```
DC1-SW3-LEAF2#show isis neighbors

Instance  VRF      System Id        Type Interface          SNPA              State Hold time   Circuit Id
1         default  DC1-SW3-SPINE1   L2   Ethernet1          P2P               UP    26          0E
1         default  DC1-SW3-SPINE2   L2   Ethernet2          P2P               UP    27          0E

DC1-SW3-LEAF2#show bfd peers
VRF name: default
-----------------
DstAddr       MyDisc    YourDisc  Interface/Transport    Type           LastUp
--------- ----------- ----------- -------------------- ------- ----------------
10.0.1.2  3628520478   922459779        Ethernet1(13)  normal   04/28/26 19:22
10.0.1.8  3070056366  4239833456        Ethernet2(14)  normal   04/28/26 20:23

   LastDown            LastDiag    State
-------------- ------------------- -----
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
```

```
DC1-SW3-LEAF3#show isis neighbors

Instance  VRF      System Id        Type Interface          SNPA              State Hold time   Circuit Id
1         default  DC1-SW3-SPINE1   L2   Ethernet1          P2P               UP    29          0F
1         default  DC1-SW3-SPINE2   L2   Ethernet2          P2P               UP    29          0F

DC1-SW3-LEAF3#show bfd peers
VRF name: default
-----------------
DstAddr        MyDisc    YourDisc  Interface/Transport    Type          LastUp
---------- ----------- ----------- -------------------- ------- ---------------
10.0.1.4   1028976166  1905081756        Ethernet1(13)  normal  04/28/26 17:46
10.0.1.10  3602833428   451113844        Ethernet2(14)  normal  04/28/26 20:23

   LastDown            LastDiag    State
-------------- ------------------- -----
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
```

Все соседства находятся в состоянии UP, проверяем IS-IS Database (для компактности примет только с DC1-SW3-SPINE1)

```
DC1-SW3-SPINE1#show isis database detail

IS-IS Instance: 1 VRF: default
  IS-IS Level 2 Link State Database
    LSPID                   Seq Num  Cksum  Life Length IS Flags
    DC1-SW3-SPINE1.00-00         45  55489   436    173 L2 <>
      LSP generation remaining wait time: 0 ms
      Time remaining until refresh: 136 s
      NLPID: 0xCC(IPv4)
      Hostname: DC1-SW3-SPINE1
      Authentication mode: MD5 Length: 17
      Area addresses: 49.0001
      Interface address: 10.0.1.4
      Interface address: 10.0.1.2
      Interface address: 10.0.1.0
      Interface address: 10.0.0.1
      IS Neighbor          : DC1-SW3-LEAF1.00    Metric: 400
      IS Neighbor          : DC1-SW3-LEAF2.00    Metric: 400
      IS Neighbor          : DC1-SW3-LEAF3.00    Metric: 400
      Reachability         : 10.0.1.4/31 Metric: 400 Type: 1 Up
      Reachability         : 10.0.1.2/31 Metric: 400 Type: 1 Up
      Reachability         : 10.0.1.0/31 Metric: 400 Type: 1 Up
      Reachability         : 10.0.0.1/32 Metric: 10 Type: 1 Up
      Router Capabilities: Router Id: 10.0.0.1 Flags: []
        Area leader priority: 250 algorithm: 0

    DC1-SW3-SPINE2.00-00         53  11858   923    173 L2 <>
      Remaining lifetime received: 1199 s Modified to: 1200 s
      NLPID: 0xCC(IPv4)
      Hostname: DC1-SW3-SPINE2
      Authentication mode: MD5 Length: 17
      Area addresses: 49.0001
      Interface address: 10.0.0.2
      Interface address: 10.0.1.10
      Interface address: 10.0.1.8
      Interface address: 10.0.1.6
      IS Neighbor          : DC1-SW3-LEAF3.00    Metric: 400
      IS Neighbor          : DC1-SW3-LEAF1.00    Metric: 400
      IS Neighbor          : DC1-SW3-LEAF2.00    Metric: 400
      Reachability         : 10.0.0.2/32 Metric: 10 Type: 1 Up
      Reachability         : 10.0.1.10/31 Metric: 400 Type: 1 Up
      Reachability         : 10.0.1.8/31 Metric: 400 Type: 1 Up
      Reachability         : 10.0.1.6/31 Metric: 400 Type: 1 Up
      Router Capabilities: Router Id: 10.0.0.2 Flags: []
        Area leader priority: 250 algorithm: 0

    DC1-SW3-LEAF1.00-00          37   6140  1099    148 L2 <>
      Remaining lifetime received: 1199 s Modified to: 1200 s
      NLPID: 0xCC(IPv4)
      Hostname: DC1-SW3-LEAF1
      Authentication mode: MD5 Length: 17
      Area addresses: 49.0001
      Interface address: 10.0.1.7
      Interface address: 10.0.0.3
      Interface address: 10.0.1.1
      IS Neighbor          : DC1-SW3-SPINE2.00   Metric: 400
      IS Neighbor          : DC1-SW3-SPINE1.00   Metric: 400
      Reachability         : 10.0.1.6/31 Metric: 400 Type: 1 Up
      Reachability         : 10.0.0.3/32 Metric: 10 Type: 1 Up
      Reachability         : 10.0.1.0/31 Metric: 400 Type: 1 Up
      Router Capabilities: Router Id: 10.0.0.3 Flags: []
        Area leader priority: 250 algorithm: 0

    DC1-SW3-LEAF2.00-00          34  52872  1126    148 L2 <>
      Remaining lifetime received: 1199 s Modified to: 1200 s
      NLPID: 0xCC(IPv4)
      Hostname: DC1-SW3-LEAF2
      Authentication mode: MD5 Length: 17
      Area addresses: 49.0001
      Interface address: 10.0.0.4
      Interface address: 10.0.1.9
      Interface address: 10.0.1.3
      IS Neighbor          : DC1-SW3-SPINE2.00   Metric: 400
      IS Neighbor          : DC1-SW3-SPINE1.00   Metric: 400
      Reachability         : 10.0.0.4/32 Metric: 10 Type: 1 Up
      Reachability         : 10.0.1.8/31 Metric: 400 Type: 1 Up
      Reachability         : 10.0.1.2/31 Metric: 400 Type: 1 Up
      Router Capabilities: Router Id: 10.0.0.4 Flags: []
        Area leader priority: 250 algorithm: 0

    DC1-SW3-LEAF3.00-00          36   9709  1099    148 L2 <>
      Remaining lifetime received: 1199 s Modified to: 1200 s
      NLPID: 0xCC(IPv4)
      Hostname: DC1-SW3-LEAF3
      Authentication mode: MD5 Length: 17
      Area addresses: 49.0001
      Interface address: 10.0.0.5
      Interface address: 10.0.1.11
      Interface address: 10.0.1.5
      IS Neighbor          : DC1-SW3-SPINE2.00   Metric: 400
      IS Neighbor          : DC1-SW3-SPINE1.00   Metric: 400
      Reachability         : 10.0.0.5/32 Metric: 10 Type: 1 Up
      Reachability         : 10.0.1.10/31 Metric: 400 Type: 1 Up
      Reachability         : 10.0.1.4/31 Metric: 400 Type: 1 Up
      Router Capabilities: Router Id: 10.0.0.5 Flags: []
        Area leader priority: 250 algorithm: 0
```

В IS-IS DB вся необходимая информация о соседях и доступных сетях присутствует, проверяем таблицу маршрутизации протокола IS-IS

```
DC1-SW3-SPINE1#show ip route isis detail

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

 I L2     10.0.0.2/32 [115/810] via 10.0.1.1, Ethernet1 DC1-SW3-LEAF1::Eth1
                                via 10.0.1.3, Ethernet2 DC1-SW3-LEAF2::Eth1
                                via 10.0.1.5, Ethernet3 DC1-SW3-LEAF3::Eth1
 I L2     10.0.0.3/32 [115/410] via 10.0.1.1, Ethernet1 DC1-SW3-LEAF1::Eth1
 I L2     10.0.0.4/32 [115/410] via 10.0.1.3, Ethernet2 DC1-SW3-LEAF2::Eth1
 I L2     10.0.0.5/32 [115/410] via 10.0.1.5, Ethernet3 DC1-SW3-LEAF3::Eth1
 I L2     10.0.1.6/31 [115/800] via 10.0.1.1, Ethernet1 DC1-SW3-LEAF1::Eth1
 I L2     10.0.1.8/31 [115/800] via 10.0.1.3, Ethernet2 DC1-SW3-LEAF2::Eth1
 I L2     10.0.1.10/31 [115/800] via 10.0.1.5, Ethernet3 DC1-SW3-LEAF3::Eth1
```

```
DC1-SW3-SPINE2#show ip route isis detail

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

 I L2     10.0.0.1/32 [115/810] via 10.0.1.7, Ethernet1 DC1-SW3-LEAF1::Eth2
                                via 10.0.1.9, Ethernet2 DC1-SW3-LEAF2::Eth2
                                via 10.0.1.11, Ethernet3 DC1-SW3-LEAF3::Eth2
 I L2     10.0.0.3/32 [115/410] via 10.0.1.7, Ethernet1 DC1-SW3-LEAF1::Eth2
 I L2     10.0.0.4/32 [115/410] via 10.0.1.9, Ethernet2 DC1-SW3-LEAF2::Eth2
 I L2     10.0.0.5/32 [115/410] via 10.0.1.11, Ethernet3 DC1-SW3-LEAF3::Eth2
 I L2     10.0.1.0/31 [115/800] via 10.0.1.7, Ethernet1 DC1-SW3-LEAF1::Eth2
 I L2     10.0.1.2/31 [115/800] via 10.0.1.9, Ethernet2 DC1-SW3-LEAF2::Eth2
 I L2     10.0.1.4/31 [115/800] via 10.0.1.11, Ethernet3 DC1-SW3-LEAF3::Eth2
```

```
DC1-SW3-LEAF1#show ip route isis detail

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

 I L2     10.0.0.1/32 [115/410] via 10.0.1.0, Ethernet1 DC1-SW3-SPINE1::Eth1
 I L2     10.0.0.2/32 [115/410] via 10.0.1.6, Ethernet2 DC1-SW3-SPINE2::Eth1
 I L2     10.0.0.4/32 [115/810] via 10.0.1.0, Ethernet1 DC1-SW3-SPINE1::Eth1
                                via 10.0.1.6, Ethernet2 DC1-SW3-SPINE2::Eth1
 I L2     10.0.0.5/32 [115/810] via 10.0.1.0, Ethernet1 DC1-SW3-SPINE1::Eth1
                                via 10.0.1.6, Ethernet2 DC1-SW3-SPINE2::Eth1
 I L2     10.0.1.2/31 [115/800] via 10.0.1.0, Ethernet1 DC1-SW3-SPINE1::Eth1
 I L2     10.0.1.4/31 [115/800] via 10.0.1.0, Ethernet1 DC1-SW3-SPINE1::Eth1
 I L2     10.0.1.8/31 [115/800] via 10.0.1.6, Ethernet2 DC1-SW3-SPINE2::Eth1
 I L2     10.0.1.10/31 [115/800] via 10.0.1.6, Ethernet2 DC1-SW3-SPINE2::Eth1
```

```
DC1-SW3-LEAF2#show ip route isis detail

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

 I L2     10.0.0.1/32 [115/410] via 10.0.1.2, Ethernet1 DC1-SW3-SPINE1::Eth2
 I L2     10.0.0.2/32 [115/410] via 10.0.1.8, Ethernet2 DC1-SW3-SPINE2::Eth2
 I L2     10.0.0.3/32 [115/810] via 10.0.1.2, Ethernet1 DC1-SW3-SPINE1::Eth2
                                via 10.0.1.8, Ethernet2 DC1-SW3-SPINE2::Eth2
 I L2     10.0.0.5/32 [115/810] via 10.0.1.2, Ethernet1 DC1-SW3-SPINE1::Eth2
                                via 10.0.1.8, Ethernet2 DC1-SW3-SPINE2::Eth2
 I L2     10.0.1.0/31 [115/800] via 10.0.1.2, Ethernet1 DC1-SW3-SPINE1::Eth2
 I L2     10.0.1.4/31 [115/800] via 10.0.1.2, Ethernet1 DC1-SW3-SPINE1::Eth2
 I L2     10.0.1.6/31 [115/800] via 10.0.1.8, Ethernet2 DC1-SW3-SPINE2::Eth2
 I L2     10.0.1.10/31 [115/800] via 10.0.1.8, Ethernet2 DC1-SW3-SPINE2::Eth2
```

```
DC1-SW3-LEAF3#show ip route isis detail

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

 I L2     10.0.0.1/32 [115/410] via 10.0.1.4, Ethernet1 DC1-SW3-SPINE1::Eth3
 I L2     10.0.0.2/32 [115/410] via 10.0.1.10, Ethernet2 DC1-SW3-SPINE2::Eth3
 I L2     10.0.0.3/32 [115/810] via 10.0.1.4, Ethernet1 DC1-SW3-SPINE1::Eth3
                                via 10.0.1.10, Ethernet2 DC1-SW3-SPINE2::Eth3
 I L2     10.0.0.4/32 [115/810] via 10.0.1.4, Ethernet1 DC1-SW3-SPINE1::Eth3
                                via 10.0.1.10, Ethernet2 DC1-SW3-SPINE2::Eth3
 I L2     10.0.1.0/31 [115/800] via 10.0.1.4, Ethernet1 DC1-SW3-SPINE1::Eth3
 I L2     10.0.1.2/31 [115/800] via 10.0.1.4, Ethernet1 DC1-SW3-SPINE1::Eth3
 I L2     10.0.1.6/31 [115/800] via 10.0.1.10, Ethernet2 DC1-SW3-SPINE2::Eth3
 I L2     10.0.1.8/31 [115/800] via 10.0.1.10, Ethernet2 DC1-SW3-SPINE2::Eth3
```

Доступность адресов lo через равноценные линки имеется

---

Далее проверяем L3 связность до между LO

*DC1-SW3-LEAF1 <> DC1-SW3-LEAF3*

```
DC1-SW3-LEAF1#ping 10.0.0.5 size 9000 df-bit source lo0
PING 10.0.0.5 (10.0.0.5) from 10.0.0.3 : 8972(9000) bytes of data.
8980 bytes from 10.0.0.5: icmp_seq=1 ttl=63 time=12.1 ms
8980 bytes from 10.0.0.5: icmp_seq=2 ttl=63 time=11.5 ms
8980 bytes from 10.0.0.5: icmp_seq=3 ttl=63 time=14.9 ms
8980 bytes from 10.0.0.5: icmp_seq=4 ttl=63 time=11.0 ms
8980 bytes from 10.0.0.5: icmp_seq=5 ttl=63 time=9.55 ms

--- 10.0.0.5 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 55ms
rtt min/avg/max/mdev = 9.558/11.855/14.915/1.762 ms, pipe 2, ipg/ewma 13.903/11.933 ms
```

*DC1-SW3-LEAF3 <> DC1-SW3-LEAF2*

```
DC1-SW3-LEAF3#ping 10.0.0.4 size 9000 df-bit source lo0
PING 10.0.0.4 (10.0.0.4) from 10.0.0.5 : 8972(9000) bytes of data.
8980 bytes from 10.0.0.4: icmp_seq=1 ttl=63 time=11.4 ms
8980 bytes from 10.0.0.4: icmp_seq=2 ttl=63 time=9.40 ms
8980 bytes from 10.0.0.4: icmp_seq=3 ttl=63 time=13.0 ms
8980 bytes from 10.0.0.4: icmp_seq=4 ttl=63 time=14.2 ms
8980 bytes from 10.0.0.4: icmp_seq=5 ttl=63 time=11.0 ms

--- 10.0.0.4 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 51ms
rtt min/avg/max/mdev = 9.402/11.843/14.257/1.687 ms, pipe 2, ipg/ewma 12.762/11.689 ms
```