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

## Настройка клиентов (VNI, коммутация между клиентами)

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

---

## Проверка