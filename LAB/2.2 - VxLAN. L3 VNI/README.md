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

**Symmetric IRB**
Для клиента CST1 помимо l2 cвязности в VLAN 100,200 (сервисы 1,2) добавляется l3 связность (сервис 3) между его l2 доменами.

**Asymmetric IRB**
Для клиента CST2 добавляется второй сервис VLAN 400 (сервис 2) и так же l3 связность между широковещательными доменами. 

**Anycast GW**
LEAF коммутаторы будут выступать в качестве шлюзов для конечных хостов клиента. 
IP и MAC адрес шлюза будет единый для всех LEAF участвующих в сервисe. Для конечных хостов это будет значить, что при миграции на другой LEAF у них не изменятся атрибуты шлюза.



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
| 5 | [cst:2][svc:2] | vlan 400 |

| RD | RT | Service | Description |
| --- | --- | --- | --- |
| 1 | 1 | [cst:1][svc:1][svc:2] | vlan-aware-bundle CST1 |
| 2 | 2 | [cst:2][svc:1] | vlan 300 |
| 3 | 3 | [cst:1][svc:3] | vrf CST1 |
| 4 | 4 | [cst:1][svc:2] | vlan 400 |



# Настройка клиентов (VNI, маршрутизация между клиентами)

> в качестве примера LEAF выступает DC1-SW3-LEAF1

```
!
vlan 100
   name [cvlan:100][cst:1][svc:1]
!
vlan 200
   name [cvlan:200][cst:1][svc:2]
!
vlan 300
   name [svlan:300][cst:2][svc:1]
!
vlan 400
   name [ctag:400][cst:1][svc:2]
!
vrf instance CST1
   description [cst:1]#SYMMETRIC-IRB#
!
vrf instance CST2
   description [cst:2]#ASYMMETRIC-IRB#
!
interface Vlan100
   vrf CST1
   ip address virtual 192.168.100.1/24
!
interface Vlan200
   vrf CST1
   ip address virtual 192.168.200.1/24
!
interface Vlan300
   vrf CST2
   ip address virtual 192.168.33.1/24
!
interface Vlan400
   vrf CST2
   ip address virtual 192.168.44.1/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 100 vni 1
   vxlan vlan 200 vni 2
   vxlan vlan 300 vni 3
   vxlan vlan 400 vni 5
   vxlan vrf CST1 vni 4
!
ip virtual-router mac-address 00:00:00:00:11:11
!
ip routing
ip routing vrf CST1
ip routing vrf CST2
!
router bgp 64513
   !
   vlan 300
      rd 10.0.0.3:2
      route-target both 64512:2
      redistribute learned
   !
   vlan 400
      rd 10.0.0.3:4
      route-target both 64512:4
      redistribute learned
   !
   vlan-aware-bundle CST1
      rd 10.0.0.3:1
      route-target both 64512:1
      redistribute learned
      vlan 100,200
   !
   vrf CST1
      rd 10.0.0.3:3
      route-target import evpn 64512:3
      route-target export evpn 64512:3
      redistribute connected
!
end
```

аналогичным образом настроен DC1-SW3-LEAF2,3

---

# Проверка

проверку EVPN, interface VxLAN и т.д. не осуществляем т.к. с прошлой работы ничего не поменялось. 



## VxLAN VNI

для симметричного IRB используется транзитный VLAN (у Arista по умолчанию задается из динамического пула)

```
DC1-SW3-LEAF1#show vxlan vni
VNI to VLAN Mapping for Vxlan1
VNI       VLAN       Source       Interface       802.1Q Tag
--------- ---------- ------------ --------------- ----------
1         100        static       Ethernet8       untagged
                                  Vxlan1          100
2         200        static       Ethernet6       untagged
                                  Vxlan1          200
3         300        static       Ethernet7       untagged
                                  Vxlan1          300
5         400        static       Vxlan1          400

VNI to dynamic VLAN Mapping for Vxlan1
VNI       VLAN       VRF        Source
--------- ---------- ---------- ------------
4         4094       CST1       evpn
```



## Связность между конечными хостами в разных VLAN

**VRF CST1 (VLAN 100,200)** 

**DC1-SR-NODE1(DC1-SW3-LEAF1) <> DC1-SR-NODE3(DC1-SW3-LEAF3)**

```
DC1-SR-NODE1> show ip

NAME        : DC1-SR-NODE1[1]
IP/MASK     : 192.168.100.11/24
GATEWAY     : 192.168.100.1
DNS         : 
MAC         : 00:50:79:66:68:01
LPORT       : 20000
RHOST:PORT  : 127.0.0.1:30000
MTU         : 1500

DC1-SR-NODE1> ping 192.168.200.3

84 bytes from 192.168.200.3 icmp_seq=1 ttl=62 time=21.932 ms
84 bytes from 192.168.200.3 icmp_seq=2 ttl=62 time=23.492 ms
84 bytes from 192.168.200.3 icmp_seq=3 ttl=62 time=22.573 ms
84 bytes from 192.168.200.3 icmp_seq=4 ttl=62 time=21.871 ms
84 bytes from 192.168.200.3 icmp_seq=5 ttl=62 time=22.566 ms
```

**VRF CST2 (VLAN 300,400)** 

**DC1-SR-NODE5(DC1-SW3-LEAF1) <> DC1-SR-NODE7(DC1-SW3-LEAF2)**

```
DC1-SR-NODE5> show ip

NAME        : DC1-SR-NODE5[1]
IP/MASK     : 192.168.33.5/24
GATEWAY     : 192.168.33.1
DNS         : 
MAC         : 00:50:79:66:68:05
LPORT       : 20000
RHOST:PORT  : 127.0.0.1:30000
MTU         : 1500

DC1-SR-NODE5> ping 192.168.44.2

84 bytes from 192.168.44.2 icmp_seq=1 ttl=63 time=457.962 ms
84 bytes from 192.168.44.2 icmp_seq=2 ttl=63 time=24.641 ms
84 bytes from 192.168.44.2 icmp_seq=3 ttl=63 time=22.213 ms
84 bytes from 192.168.44.2 icmp_seq=4 ttl=63 time=21.583 ms
84 bytes from 192.168.44.2 icmp_seq=5 ttl=63 time=22.130 ms
```

связность между внутри VRF между разными l2 сегментами присутствует



## Type 2 routes

**DC1-SW3-LEAF1**

```
DC1-SW3-LEAF1#show bgp evpn route-type mac-ip vni 4 detail
BGP routing table information for VRF default
Router identifier 10.0.0.3, local AS number 64513
BGP routing table entry for mac-ip 1 0050.7966.6801 192.168.100.11, Route Distinguisher: 10.0.0.3:1
 Paths: 1 available
  Local
    - from - (0.0.0.0)
      Origin IGP, metric -, localpref -, weight 0, tag 0, valid, local, best
      Extended Community: Route-Target-AS:64512:1 Route-Target-AS:64512:3 TunnelEncap:tunnelTypeVxlan
      VNI: 1 L3 VNI: 4 ESI: 0000:0000:0000:0000:0000
BGP routing table entry for mac-ip 1 0050.7966.6804 192.168.100.4, Route Distinguisher: 10.0.0.5:1
 Paths: 2 available
  64512 64515
    10.0.0.5 from 10.0.0.1 (10.0.0.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:64512:1 Route-Target-AS:64512:3 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:6d:26:f2:3c:e0
      VNI: 1 L3 VNI: 4 ESI: 0000:0000:0000:0000:0000
  64512 64515
    10.0.0.5 from 10.0.0.2 (10.0.0.2)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:64512:1 Route-Target-AS:64512:3 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:6d:26:f2:3c:e0
      VNI: 1 L3 VNI: 4 ESI: 0000:0000:0000:0000:0000
BGP routing table entry for mac-ip 2 0050.7966.6803 192.168.200.3, Route Distinguisher: 10.0.0.5:1
 Paths: 2 available
  64512 64515
    10.0.0.5 from 10.0.0.2 (10.0.0.2)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:64512:1 Route-Target-AS:64512:3 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:6d:26:f2:3c:e0
      VNI: 2 L3 VNI: 4 ESI: 0000:0000:0000:0000:0000
  64512 64515
    10.0.0.5 from 10.0.0.1 (10.0.0.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:64512:1 Route-Target-AS:64512:3 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:6d:26:f2:3c:e0
      VNI: 2 L3 VNI: 4 ESI: 0000:0000:0000:0000:0000
BGP routing table entry for mac-ip 2 0050.7966.6806 192.168.200.6, Route Distinguisher: 10.0.0.3:1
 Paths: 1 available
  Local
    - from - (0.0.0.0)
      Origin IGP, metric -, localpref -, weight 0, tag 0, valid, local, best
      Extended Community: Route-Target-AS:64512:1 Route-Target-AS:64512:3 TunnelEncap:tunnelTypeVxlan
      VNI: 2 L3 VNI: 4 ESI: 0000:0000:0000:0000:0000
```

видно что в маршрутах присутствует L3 VNI (4) помимо L2 VNI (1,2)

```
DC1-SW3-LEAF1#show bgp evpn route-type mac-ip vni 5
BGP routing table information for VRF default
Router identifier 10.0.0.3, local AS number 64513
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.0.0.4:4 mac-ip 0050.7966.680c
                                 10.0.0.4              -       100     0       64512 64514 i
 *  ec    RD: 10.0.0.4:4 mac-ip 0050.7966.680c
                                 10.0.0.4              -       100     0       64512 64514 i
 * >Ec    RD: 10.0.0.4:4 mac-ip 0050.7966.680c 192.168.44.2
                                 10.0.0.4              -       100     0       64512 64514 i
 *  ec    RD: 10.0.0.4:4 mac-ip 0050.7966.680c 192.168.44.2
                                 10.0.0.4              -       100     0       64512 64514 i


DC1-SW3-LEAF1#show bgp evpn route-type mac-ip vni 3
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
 * >      RD: 10.0.0.3:2 mac-ip 0050.7966.6805 192.168.33.5
                                 -                     -       -       0       i

для ассиметричного IRB маршрутизация на основании mac+ip маршрутов в L2VNI
```



## Дамп трафика

Проверим инкапсуляцию для симметричного и ассиметричного IRB

**VRF CST1 - Symmetric IRB**
![Symmetric IRB](../2.2%20-%20VxLAN.%20L3%20VNI/Attached%20files/DUMP_LEAF1_EHT8_SIM-IRB.png)

трафик упаковывается в L3VNI - 4


**VRF CST2 - Asymmetric IRB**
![Asymmetric IRB](../2.2%20-%20VxLAN.%20L3%20VNI/Attached%20files/DUMP_LEAF1_ETH8_ASIM-IRB.png)

трафик упаковывается в L2VNI - 5