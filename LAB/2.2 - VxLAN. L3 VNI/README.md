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
