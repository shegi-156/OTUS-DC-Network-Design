# Построение OVERLAY сети (VxLAN. EVPN L2)

## Цель работы 

Настроить Коммутацию в рамках Overlay между клиентами

## План работы

- Настроить клиентов (VNI, коммутацию между клиентами)
- Проверить связность между клиентами

## Тестовая среда 
Схема сети в [PnetLab](../2.1%20-%20VxLAN.%20L2%20VNI/Attached%20files/LAB_2_1%20EVPN%20L2.unl)

![Схема сети](../2.1%20-%20VxLAN.%20L2%20VNI/Attached%20files/LAB_2_1%20EVPN%20L2.png)
Конфигурации сетевых элементов:

[DC1-SW3-LEAF1](../2.1%20-%20VxLAN.%20L2%20VNI/CFG/DC1-SW3-LEAF1.txt)

[DC1-SW3-LEAF2](../2.1%20-%20VxLAN.%20L2%20VNI/CFG/DC1-SW3-LEAF2.txt)

[DC1-SW3-LEAF3](../2.1%20-%20VxLAN.%20L2%20VNI/CFG/DC1-SW3-LEAF3.txt)

[DC1-SW3-SPINE1](../2.1%20-%20VxLAN.%20L2%20VNI/CFG/DC1-SW3-SPINE1.txt)

[DC1-SW3-SPINE2](../2.1%20-%20VxLAN.%20L2%20VNI/CFG/DC1-SW3-SPINE2.txt)



## Описание лабораторного стенда

В рамках лабораторной работы два клиента CST1 и CST2

для клиента CST1 (два сервиса - VLAN 100,200) используем VLAN Aware Bundle Service, 
для клиента CST2 (один сервис - VLAN 300) используем VLAN Based Service

Принято решение Underlay оставить на eBGP из предыдущей лабораторной работы

*BFD отключен намеренно, в рамках тестовой среды не принципиально его наличие, а при большом количестве устройств в PNETLAB начинают флапать BFD,BGP*

Строим EVPN BGP сессии между loopback адресами, а так же сессии в AF Rt-Filter что бы не передавать на LEAF маршруты с ненужными RT

Для Arista next-hop-unchanged - это дефолтное поведение (evpn маршруты передаются с next-hop адресами loopback коммутаторов LEAF)

Так же для Arista дефолтным является поведение отсутствия фильтрации префиксов по RT. Spine не отбрасывает префиксы с RT которые ему не известны (VRF не создан локально), т.е.
`retain route-target all` не требуется

## Учет ID

VNI = порядковые значения
RD = loopback+порядковый id
RT = SPINE-AS+порядковый id

| VNI id | Description |
| --- | --- |
| 1 | [cst:1][svc:1] |
| 2 | [cst:1][svc:2] |
| 3 | [cst:2][svc:1] |

| RD | RT | Description |
| --- | --- | --- |
| 1 | 1 | [cst:1][svc:1][svc:2] |
| 2 | 2 | [cst:2][svc:1] |

# Настройка клиентов (VNI, коммутация между клиентами)

> в качестве примера LEAF выступает DC1-SW3-LEAF1

```
vlan 100
   name [cvlan:100][cst:1][svc:1]
!
vlan 200
   name [cvlan:200][cst:1][svc:2]
!
vlan 300
   name [svlan:300][cst:2][svc:1]
!
interface Ethernet6
   description [dev:DC1-SR-NODE6][int:ETH0]
   switchport access vlan 200
!
interface Ethernet7
   description [dev:DC1-SR-NODE5][int:ETH0]
   switchport access vlan 300
!
interface Ethernet8
   description [dev:DC1-SR-NODE1][int:ETH0]
   switchport access vlan 100
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 100 vni 1
   vxlan vlan 200 vni 2
   vxlan vlan 300 vni 3
!
router bgp 64513
   neighbor PG_SPINE_EVPN peer group
   neighbor PG_SPINE_EVPN remote-as 64512
   neighbor PG_SPINE_EVPN update-source Loopback0
   neighbor PG_SPINE_EVPN ebgp-multihop 3
   neighbor PG_SPINE_EVPN send-community extended
   neighbor 10.0.0.1 peer group PG_SPINE_EVPN
   neighbor 10.0.0.1 description [dev:DC1-SW3-SPINE1]#EVPN#
   neighbor 10.0.0.2 peer group PG_SPINE_EVPN
   neighbor 10.0.0.2 description [dev:DC1-SW3-SPINE2]#EVPN#
   !
   vlan 300
      rd 10.0.0.3:2
      route-target both 64512:2
      redistribute learned
   !
   vlan-aware-bundle CST1
      rd 10.0.0.3:1
      route-target both 64512:1
      redistribute learned
      vlan 100,200
   !
   address-family evpn
      neighbor PG_SPINE_EVPN activate
!
   address-family rt-membership
      neighbor PG_SPINE_EVPN activate
!
end
```

аналогичным образом настроен DC1-SW3-LEAF2,3

> в качестве примера SPINE выступает DC1-SW3-SPINE1

```
router bgp 64512
   bgp listen range 10.0.0.0/24 peer-group PG_LEAF_EVPN peer-filter PF_LEAF
   neighbor PG_LEAF_EVPN peer group
   neighbor PG_LEAF_EVPN next-hop-unchanged
   neighbor PG_LEAF_EVPN update-source Loopback0
   neighbor PG_LEAF_EVPN ebgp-multihop 3
   neighbor PG_LEAF_EVPN send-community extended
   !
   address-family evpn
      neighbor PG_LEAF_EVPN activate
   !
   address-family rt-membership
      neighbor PG_LEAF_EVPN activate
!
end
```

аналогичным образом настроен DC1-SW3-SPINE2

---

# Проверка

## BGP EVPN

Проверяем состояние BGP сессий в AF evpn/rt-membership на SPINE 1,2

**DC1-SW3-SPINE1**

```
DC1-SW3-SPINE1#show bgp summary
BGP summary information for VRF default
Router identifier 10.0.0.1, local AS number 64512
Neighbor          AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc
-------- ----------- ------------- ----------------------- -------------- ---------- ----------
10.0.0.3       64513 Established   IPv4 RtMembership       Negotiated              2          2
10.0.0.3       64513 Established   L2VPN EVPN              Negotiated              3          3
10.0.0.4       64514 Established   IPv4 RtMembership       Negotiated              1          1
10.0.0.4       64514 Established   L2VPN EVPN              Negotiated              1          1
10.0.0.5       64515 Established   IPv4 RtMembership       Negotiated              1          1
10.0.0.5       64515 Established   L2VPN EVPN              Negotiated              2          2
10.0.1.1       64513 Established   IPv4 Unicast            Negotiated              1          1
10.0.1.3       64514 Established   IPv4 Unicast            Negotiated              1          1
10.0.1.5       64515 Established   IPv4 Unicast            Negotiated              1          1
```

**DC1-SW3-SPINE2**

```
DC1-SW3-SPINE2#show bgp summary
BGP summary information for VRF default
Router identifier 10.0.0.2, local AS number 64512
Neighbor           AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc
--------- ----------- ------------- ----------------------- -------------- ---------- ----------
10.0.0.3        64513 Established   IPv4 RtMembership       Negotiated              2          2
10.0.0.3        64513 Established   L2VPN EVPN              Negotiated              3          3
10.0.0.4        64514 Established   IPv4 RtMembership       Negotiated              1          1
10.0.0.4        64514 Established   L2VPN EVPN              Negotiated              1          1
10.0.0.5        64515 Established   IPv4 RtMembership       Negotiated              1          1
10.0.0.5        64515 Established   L2VPN EVPN              Negotiated              2          2
10.0.1.7        64513 Established   IPv4 Unicast            Negotiated              1          1
10.0.1.9        64514 Established   IPv4 Unicast            Negotiated              1          1
10.0.1.11       64515 Established   IPv4 Unicast            Negotiated              1          1
```

сессии в established для всех LEAF коммутаторов



## int VxLAN

Проверяем состояние интерфейсов VxLAN 

**DC1-SW3-LEAF1**

```
DC1-SW3-LEAF1#show interfaces vxlan 1
Vxlan1 is up, line protocol is up (connected)
  Hardware is Vxlan
  Source interface is Loopback0 and is active with 10.0.0.3
  Listening on UDP port 4789
  Replication/Flood Mode is headend with Flood List Source: EVPN
  Remote MAC learning via EVPN
  VNI mapping to VLANs
  Static VLAN to VNI mapping is
    [100, 1]          [200, 2]          [300, 3]
  Note: All Dynamic VLANs used by VCS are internal VLANs.
        Use 'show vxlan vni' for details.
  Static VRF to VNI mapping is not configured
  Headend replication flood vtep list is:
   100 10.0.0.5
   200 10.0.0.5
   300 10.0.0.4
```

**DC1-SW3-LEAF**

```
DC1-SW3-LEAF2#show interfaces vxlan 1
Vxlan1 is up, line protocol is up (connected)
  Hardware is Vxlan
  Source interface is Loopback0 and is active with 10.0.0.4
  Listening on UDP port 4789
  Replication/Flood Mode is headend with Flood List Source: EVPN
  Remote MAC learning via EVPN
  VNI mapping to VLANs
  Static VLAN to VNI mapping is
    [300, 3]
  Note: All Dynamic VLANs used by VCS are internal VLANs.
        Use 'show vxlan vni' for details.
  Static VRF to VNI mapping is not configured
  Headend replication flood vtep list is:
   300 10.0.0.3
  Shared Router MAC is 0000.0000.0000
```

**DC1-SW3-LEAF3**

```
DC1-SW3-LEAF3#show interfaces vxlan 1
Vxlan1 is up, line protocol is up (connected)
  Hardware is Vxlan
  Source interface is Loopback0 and is active with 10.0.0.5
  Listening on UDP port 4789
  Replication/Flood Mode is headend with Flood List Source: EVPN
  Remote MAC learning via EVPN
  VNI mapping to VLANs
  Static VLAN to VNI mapping is
    [100, 1]          [200, 2]
  Note: All Dynamic VLANs used by VCS are internal VLANs.
        Use 'show vxlan vni' for details.
  Static VRF to VNI mapping is not configured
  Headend replication flood vtep list is:
   100 10.0.0.3
   200 10.0.0.3
  Shared Router MAC is 0000.0000.0000
```

Видим что интерфейсы в состоянии UP, видим ассоциацию VLAN с VNI, а так же что репликация происходит на все VTEP участвующие в mac vrf



## Type 3 routes

Проверяем маршруты type3 на LEAF

**DC1-SW3-LEAF1**

```
DC1-SW3-LEAF1#show bgp evpn route-type imet
BGP routing table information for VRF default
Router identifier 10.0.0.3, local AS number 64513
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.0.0.3:2 imet 10.0.0.3
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.0.4:2 imet 10.0.0.4
                                 10.0.0.4              -       100     0       64512 64514 i
 *  ec    RD: 10.0.0.4:2 imet 10.0.0.4
                                 10.0.0.4              -       100     0       64512 64514 i
 * >      RD: 10.0.0.3:1 imet 1 10.0.0.3
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.0.5:1 imet 1 10.0.0.5
                                 10.0.0.5              -       100     0       64512 64515 i
 *  ec    RD: 10.0.0.5:1 imet 1 10.0.0.5
                                 10.0.0.5              -       100     0       64512 64515 i
 * >      RD: 10.0.0.3:1 imet 2 10.0.0.3
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.0.5:1 imet 2 10.0.0.5
                                 10.0.0.5              -       100     0       64512 64515 i
 *  ec    RD: 10.0.0.5:1 imet 2 10.0.0.5
                                 10.0.0.5              -       100     0       64512 64515 i```
```

**DC1-SW3-LEAF2**

```
DC1-SW3-LEAF2#show bgp evpn route-type imet
BGP routing table information for VRF default
Router identifier 10.0.0.4, local AS number 64514
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.0.0.3:2 imet 10.0.0.3
                                 10.0.0.3              -       100     0       64512 64513 i
 *  ec    RD: 10.0.0.3:2 imet 10.0.0.3
                                 10.0.0.3              -       100     0       64512 64513 i
 * >      RD: 10.0.0.4:2 imet 10.0.0.4

```

**DC1-SW3-LEAF3**

```
DC1-SW3-LEAF3#show bgp evpn route-type imet
BGP routing table information for VRF default
Router identifier 10.0.0.5, local AS number 64515
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.0.0.3:1 imet 1 10.0.0.3
                                 10.0.0.3              -       100     0       64512 64513 i
 *  ec    RD: 10.0.0.3:1 imet 1 10.0.0.3
                                 10.0.0.3              -       100     0       64512 64513 i
 * >      RD: 10.0.0.5:1 imet 1 10.0.0.5
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.0.3:1 imet 2 10.0.0.3
                                 10.0.0.3              -       100     0       64512 64513 i
 *  ec    RD: 10.0.0.3:1 imet 2 10.0.0.3
                                 10.0.0.3              -       100     0       64512 64513 i
 * >      RD: 10.0.0.5:1 imet 2 10.0.0.5
                                 -                     -       -       0       i

```

Видим что в качестве NH указаны lo LEAF коммутаторов, это говорит о том что NH не подменяется на SPINE и маршруты передаются с RT которых нет в локальном импорте SPINE (vrf не настроен)



## RT-filter

так же видим что на LEAF анонсируются только маршруты для тех mac vrf которые настроены на них локально

**для LEAF2 (VNI 3; RT 64512:2)**

```
DC1-SW3-SPINE1#show bgp neighbors 10.0.0.4 evpn advertised-routes detail
BGP routing table information for VRF default
Router identifier 10.0.0.1, local AS number 64512
Update wait-install is disabled
BGP routing table entry for imet 10.0.0.3, Route Distinguisher: 10.0.0.3:2
 Paths: 1 available
  64512 64513
    10.0.0.3 from 10.0.0.3 (10.0.0.3)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, best
      Extended Community: Route-Target-AS:64512:2 TunnelEncap:tunnelTypeVxlan
      VNI: 3
      PMSI Tunnel: Ingress Replication, MPLS Label: 3, Leaf Information Required: false, Tunnel ID: 10.0.0.3
```

**для LEAF3 (VNI 1; RT 64512:1)**

```
DC1-SW3-SPINE2#show bgp neighbors 10.0.0.5 evpn advertised-routes detail
BGP routing table information for VRF default
Router identifier 10.0.0.2, local AS number 64512
Update wait-install is disabled
BGP routing table entry for imet 1 10.0.0.3, Route Distinguisher: 10.0.0.3:1
 Paths: 1 available
  64512 64513
    10.0.0.3 from 10.0.0.3 (10.0.0.3)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, best
      Extended Community: Route-Target-AS:64512:1 TunnelEncap:tunnelTypeVxlan
      VNI: 1
      PMSI Tunnel: Ingress Replication, MPLS Label: 1, Leaf Information Required: false, Tunnel ID: 10.0.0.3
BGP routing table entry for imet 2 10.0.0.3, Route Distinguisher: 10.0.0.3:1
 Paths: 1 available
  64512 64513
    10.0.0.3 from 10.0.0.3 (10.0.0.3)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, best
      Extended Community: Route-Target-AS:64512:1 TunnelEncap:tunnelTypeVxlan
      VNI: 2
      PMSI Tunnel: Ingress Replication, MPLS Label: 2, Leaf Information Required: false, Tunnel ID: 10.0.0.3
```



## Связность между конечными хостами

**DC1-SR-NODE1(DC1-SW3-LEAF1) <> DC1-SR-NODE4(DC1-SW3-LEAF3)**

```
DC1-SR-NODE1>  ping 192.168.100.4

84 bytes from 192.168.100.4 icmp_seq=1 ttl=64 time=35.913 ms
84 bytes from 192.168.100.4 icmp_seq=2 ttl=64 time=24.547 ms
84 bytes from 192.168.100.4 icmp_seq=3 ttl=64 time=19.200 ms
84 bytes from 192.168.100.4 icmp_seq=4 ttl=64 time=19.858 ms
84 bytes from 192.168.100.4 icmp_seq=5 ttl=64 time=23.864 ms
```

**DC1-SR-NODE5(DC1-SW3-LEAF1) <> DC1-SR-NODE2(DC1-SW3-LEAF2)**

```
DC1-SR-NODE5> ping 192.168.33.2

84 bytes from 192.168.33.2 icmp_seq=1 ttl=64 time=35.794 ms
84 bytes from 192.168.33.2 icmp_seq=2 ttl=64 time=20.301 ms
84 bytes from 192.168.33.2 icmp_seq=3 ttl=64 time=18.582 ms
84 bytes from 192.168.33.2 icmp_seq=4 ttl=64 time=18.065 ms
84 bytes from 192.168.33.2 icmp_seq=5 ttl=64 time=23.286 ms
```

**DC1-SR-NODE6(DC1-SW3-LEAF1) <> DC1-SR-NODE3(DC1-SW3-LEAF3)**

```
DC1-SR-NODE6> ping 192.168.200.3

84 bytes from 192.168.200.3 icmp_seq=1 ttl=64 time=43.251 ms
84 bytes from 192.168.200.3 icmp_seq=2 ttl=64 time=21.421 ms
84 bytes from 192.168.200.3 icmp_seq=3 ttl=64 time=23.007 ms
84 bytes from 192.168.200.3 icmp_seq=4 ttl=64 time=16.954 ms
84 bytes from 192.168.200.3 icmp_seq=5 ttl=64 time=18.457 ms
```

хосты пингуются в пределах своих широковещательных доменов



## Type 2 routes и MAC таблици



**DC1-SW3-LEAF1**

```
DC1-SW3-LEAF1#show bgp evpn route-type mac-ip
BGP routing table information for VRF default
Router identifier 10.0.0.3, local AS number 64513
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.0.0.4:2 mac-ip 0050.7966.6802
                                 10.0.0.4              -       100     0       64512 64514 i
 *  ec    RD: 10.0.0.4:2 mac-ip 0050.7966.6802
                                 10.0.0.4              -       100     0       64512 64514 i
 * >      RD: 10.0.0.3:2 mac-ip 0050.7966.6805
                                 -                     -       -       0       i
 * >      RD: 10.0.0.3:1 mac-ip 1 0050.7966.6801
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.0.5:1 mac-ip 1 0050.7966.6804
                                 10.0.0.5              -       100     0       64512 64515 i
 *  ec    RD: 10.0.0.5:1 mac-ip 1 0050.7966.6804
                                 10.0.0.5              -       100     0       64512 64515 i
 * >Ec    RD: 10.0.0.5:1 mac-ip 2 0050.7966.6803
                                 10.0.0.5              -       100     0       64512 64515 i
 *  ec    RD: 10.0.0.5:1 mac-ip 2 0050.7966.6803
                                 10.0.0.5              -       100     0       64512 64515 i
 * >      RD: 10.0.0.3:1 mac-ip 2 0050.7966.6806
                                 -                     -       -       0       i
```

```
DC1-SW3-LEAF1#show l2rib output detail
0050.7966.6802, VLAN 300, seq 1, pref 16, evpnDynamicRemoteMac, source: BGP
   VTEP 10.0.0.4
0050.7966.6805, VLAN 300, seq 1, pref 16, learnedDynamicMac, source: Local Dynamic
   Ethernet7
0050.7966.6801, VLAN 100, seq 1, pref 16, learnedDynamicMac, source: Local Dynamic
   Ethernet8
0050.7966.6803, VLAN 200, seq 1, pref 16, evpnDynamicRemoteMac, source: BGP
   VTEP 10.0.0.5
0050.7966.6806, VLAN 200, seq 1, pref 16, learnedDynamicMac, source: Local Dynamic
   Ethernet6
0050.7966.6804, VLAN 100, seq 1, pref 16, evpnDynamicRemoteMac, source: BGP
   VTEP 10.0.0.5
```


**DC1-SW3-LEAF2**

```
DC1-SW3-LEAF2#show bgp evpn route-type mac-ip
BGP routing table information for VRF default
Router identifier 10.0.0.4, local AS number 64514
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.0.0.4:2 mac-ip 0050.7966.6802
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.0.3:2 mac-ip 0050.7966.6805
                                 10.0.0.3              -       100     0       64512 64513 i
 *  ec    RD: 10.0.0.3:2 mac-ip 0050.7966.6805
                                 10.0.0.3              -       100     0       64512 64513 i
```

```
DC1-SW3-LEAF2#show l2rib output detail
0050.7966.6802, VLAN 300, seq 1, pref 16, learnedDynamicMac, source: Local Dynamic
   Ethernet8
0050.7966.6805, VLAN 300, seq 1, pref 16, evpnDynamicRemoteMac, source: BGP
   VTEP 10.0.0.3
```

**DC1-SW3-LEAF3**

```
DC1-SW3-LEAF3#show bgp evpn route-type mac-ip
BGP routing table information for VRF default
Router identifier 10.0.0.5, local AS number 64515
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.0.0.3:1 mac-ip 1 0050.7966.6801
                                 10.0.0.3              -       100     0       64512 64513 i
 *  ec    RD: 10.0.0.3:1 mac-ip 1 0050.7966.6801
                                 10.0.0.3              -       100     0       64512 64513 i
 * >      RD: 10.0.0.5:1 mac-ip 1 0050.7966.6804
                                 -                     -       -       0       i
 * >      RD: 10.0.0.5:1 mac-ip 2 0050.7966.6803
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.0.3:1 mac-ip 2 0050.7966.6806
                                 10.0.0.3              -       100     0       64512 64513 i
 *  ec    RD: 10.0.0.3:1 mac-ip 2 0050.7966.6806
                                 10.0.0.3              -       100     0       64512 64513 i
```

```
DC1-SW3-LEAF3#show l2rib output detail
0050.7966.6801, VLAN 100, seq 1, pref 16, evpnDynamicRemoteMac, source: BGP
   VTEP 10.0.0.3
0050.7966.6803, VLAN 200, seq 1, pref 16, learnedDynamicMac, source: Local Dynamic
   Ethernet7
0050.7966.6806, VLAN 200, seq 1, pref 16, evpnDynamicRemoteMac, source: BGP
   VTEP 10.0.0.3
0050.7966.6804, VLAN 100, seq 1, pref 16, learnedDynamicMac, source: Local Dynamic
   Ethernet8

```

Все маки изучены корректно



## Дамп трафика

запустим пинг между хостами для CST1-SVC1 (VNI 1) 

**DC1-SR-NODE1(DC1-SW3-LEAF1) <> DC1-SR-NODE4(DC1-SW3-LEAF3)**

![LEAF1 ETH8](../2.1%20-%20VxLAN.%20L2%20VNI/Attached%20files/DUMP_LEAF1_ETH8.png)

на ETH8 (UNI) интерфейсе обычный трафик

![LEAF1 ETH1](../2.1%20-%20VxLAN.%20L2%20VNI/Attached%20files/DUMP_LEAF1_ETH1.png)

на ETH1 (NNI) интерфейсе уже инкапсулирован в VxLAN