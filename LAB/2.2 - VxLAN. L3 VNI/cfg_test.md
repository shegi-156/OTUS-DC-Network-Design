### DC1-SW3-LEAF1
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname DC1-SW3-LEAF1
!
spanning-tree mode mstp
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
vrf instance CST1_SVC2
!
interface Ethernet1
   description [dev:DC1-SW3-SPINE1][int:ETH1]
   mtu 9000
   no switchport
   ip address 10.0.1.1/31
!
interface Ethernet2
   description [dev:DC1-SW3-SPINE2][int:ETH1]
   mtu 9000
   no switchport
   ip address 10.0.1.7/31
!
interface Ethernet3
!
interface Ethernet4
!
interface Ethernet5
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
interface Loopback0
   description #UNDERLAY#
   ip address 10.0.0.3/32
!
interface Management1
   description [dev:DC1-SW-MGMT1}
   ip address 172.16.0.3/24
!
interface Vlan100
   vrf CST1_SVC2
   ip address virtual 192.168.100.1/24
!
interface Vlan200
   vrf CST1_SVC2
   ip address virtual 192.168.200.1/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 100 vni 1001
   vxlan vlan 200 vni 1002
   vxlan vlan 300 vni 1003
   vxlan vrf CST1_SVC2 vni 2002
!
ip virtual-router mac-address 00:00:00:00:11:11
!
ip routing
ip routing vrf CST1_SVC2
!
router bfd
   interval 100 min-rx 100 multiplier 3 default
!
router bgp 64513
   router-id 10.0.0.3
   no bgp default ipv4-unicast
   timers bgp 3 9
   distance bgp 20 200 200
   maximum-paths 4 ecmp 4
   neighbor PG_SPINE peer group
   neighbor PG_SPINE remote-as 64512
   neighbor PG_SPINE password 7 3ZMHuSYHdBchNfOxO2nKBg==
   neighbor PG_SPINE send-community standard extended large
   neighbor PG_SPINE_EVPN peer group
   neighbor PG_SPINE_EVPN remote-as 64512
   neighbor PG_SPINE_EVPN update-source Loopback0
   neighbor PG_SPINE_EVPN ebgp-multihop 3
   neighbor PG_SPINE_EVPN send-community extended
   neighbor 10.0.0.1 peer group PG_SPINE_EVPN
   neighbor 10.0.0.1 description [dev:DC1-SW3-SPINE1]#EVPN#
   neighbor 10.0.0.2 peer group PG_SPINE_EVPN
   neighbor 10.0.0.2 description [dev:DC1-SW3-SPINE2]#EVPN#
   neighbor 10.0.1.0 peer group PG_SPINE
   neighbor 10.0.1.0 description [dev:DC1-SW3-SPINE1]#p2p#
   neighbor 10.0.1.6 peer group PG_SPINE
   neighbor 10.0.1.6 description [dev:DC1-SW3-SPINE2]#p2p#
   !
   vlan 300
      rd 10.0.0.3:2
      route-target both 1:101
      redistribute learned
   !
   vlan-aware-bundle CST1
      rd 10.0.0.3:1
      route-target both 1:100
      redistribute learned
      vlan 100,200
   !
   address-family evpn
      neighbor PG_SPINE_EVPN activate
   !
   address-family ipv4
      neighbor PG_SPINE activate
      network 10.0.0.3/32
   !
   vrf CST1_SVC2
      rd 10.0.0.3:1
      route-target import evpn 2:100
      route-target export evpn 2:100
      redistribute connected
!
end



### DC1-SW3-LEAF2
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname DC1-SW3-LEAF2
!
spanning-tree mode mstp
!
vlan 300
   name [ctag:300][sct:2][svc:1]
!
interface Ethernet1
   description [dev:DC1-SW3-SPINE1][int:ETH2]
   mtu 9000
   no switchport
   ip address 10.0.1.3/31
!
interface Ethernet2
   description [dev:DC1-SW3-SPINE2][int:ETH2]
   mtu 9000
   no switchport
   ip address 10.0.1.9/31
!
interface Ethernet3
!
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
   description [dev:DC1-SR-NODE2][int:ETH0]
   switchport access vlan 300
!
interface Loopback0
   description #UNDERLAY#
   ip address 10.0.0.4/32
!
interface Management1
   description [dev:DC1-SW-MGMT1}
   ip address 172.16.0.4/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 300 vni 1003
!
ip routing
!
router bfd
   interval 100 min-rx 100 multiplier 3 default
!
router bgp 64514
   router-id 10.0.0.4
   no bgp default ipv4-unicast
   timers bgp 3 9
   distance bgp 20 200 200
   maximum-paths 4 ecmp 4
   neighbor PG_SPINE peer group
   neighbor PG_SPINE remote-as 64512
   neighbor PG_SPINE password 7 3ZMHuSYHdBchNfOxO2nKBg==
   neighbor PG_SPINE send-community standard extended large
   neighbor PG_SPINE_EVPN peer group
   neighbor PG_SPINE_EVPN remote-as 64512
   neighbor PG_SPINE_EVPN update-source Loopback0
   neighbor PG_SPINE_EVPN ebgp-multihop 3
   neighbor PG_SPINE_EVPN send-community extended
   neighbor 10.0.0.1 peer group PG_SPINE_EVPN
   neighbor 10.0.0.1 description [dev:DC1-SW3-SPINE1]#EVPN#
   neighbor 10.0.0.2 peer group PG_SPINE_EVPN
   neighbor 10.0.0.2 description [dev:DC1-SW3-SPINE2]#EVPN#
   neighbor 10.0.1.2 peer group PG_SPINE
   neighbor 10.0.1.2 description [dev:DC1-SW3-SPINE1]#p2p#
   neighbor 10.0.1.8 peer group PG_SPINE
   neighbor 10.0.1.8 description [dev:DC1-SW3-SPINE2]#p2p#
   !
   vlan 300
      rd 10.0.0.4:2
      route-target both 1:101
      redistribute learned
   !
   address-family evpn
      neighbor PG_SPINE_EVPN activate
   !
   address-family ipv4
      neighbor PG_SPINE activate
      network 10.0.0.4/32
!
end



### DC1-SW3-LEAF3
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname DC1-SW3-LEAF3
!
spanning-tree mode mstp
!
vlan 100
   name [ctag:100][cst:1][svc:1]
!
vlan 200
   name [ctag:200][sct:1][svc:2]
!
vrf instance CST1_SVC2
!
interface Ethernet1
   description [dev:DC1-SW3-SPINE1][int:ETH3]
   mtu 9000
   no switchport
   ip address 10.0.1.5/31
!
interface Ethernet2
   description [dev:DC1-SW3-SPINE2][int:ETH3]
   mtu 9000
   no switchport
   ip address 10.0.1.11/31
!
interface Ethernet3
!
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
   description [dev:DC1-SR-NODE3][int:ETH0]
   switchport access vlan 200
!
interface Ethernet8
   description [dev:DC1-SR-NODE4][int:ETH0]
   switchport access vlan 100
!
interface Loopback0
   description #UNDERLAY#
   ip address 10.0.0.5/32
!
interface Management1
   description [dev:DC1-SW-MGMT1]
   ip address 172.16.0.5/24
!
interface Vlan100
   vrf CST1_SVC2
   ip address virtual 192.168.100.1/24
!
interface Vlan200
   vrf CST1_SVC2
   ip address virtual 192.168.200.1/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 100 vni 1001
   vxlan vlan 200 vni 1002
   vxlan vrf CST1_SVC2 vni 2002
!
ip virtual-router mac-address 00:00:00:00:11:11
!
ip routing
ip routing vrf CST1_SVC2
!
router bfd
   interval 100 min-rx 100 multiplier 3 default
!
router bgp 64515
   router-id 10.0.0.5
   no bgp default ipv4-unicast
   timers bgp 3 9
   distance bgp 20 200 200
   maximum-paths 4 ecmp 4
   neighbor PG_SPINE peer group
   neighbor PG_SPINE remote-as 64512
   neighbor PG_SPINE password 7 3ZMHuSYHdBchNfOxO2nKBg==
   neighbor PG_SPINE send-community standard extended large
   neighbor PG_SPINE_EVPN peer group
   neighbor PG_SPINE_EVPN remote-as 64512
   neighbor PG_SPINE_EVPN update-source Loopback0
   neighbor PG_SPINE_EVPN ebgp-multihop 3
   neighbor PG_SPINE_EVPN send-community extended
   neighbor 10.0.0.1 peer group PG_SPINE_EVPN
   neighbor 10.0.0.1 description [dev:DC1-SW3-SPINE1]#EVPN#
   neighbor 10.0.0.2 peer group PG_SPINE_EVPN
   neighbor 10.0.0.2 description [dev:DC1-SW3-SPINE2]#EVPN#
   neighbor 10.0.1.4 peer group PG_SPINE
   neighbor 10.0.1.4 description [dev:DC1-SW3-SPINE1]#p2p#
   neighbor 10.0.1.10 peer group PG_SPINE
   neighbor 10.0.1.10 description [dev:DC1-SW3-SPINE2]#p2p#
   !
   vlan-aware-bundle CST1
      rd 10.0.0.5:1
      route-target both 1:100
      redistribute learned
      vlan 100,200
   !
   address-family evpn
      neighbor PG_SPINE_EVPN activate
   !
   address-family ipv4
      neighbor PG_SPINE activate
      network 10.0.0.5/32
   !
   vrf CST1_SVC2
      rd 10.0.0.5:1
      route-target import evpn 2:100
      route-target export evpn 2:100
      redistribute connected
!
end



### DC1-SW3-SPINE1
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
logging monitor informational
!
hostname DC1-SW3-SPINE1
!
spanning-tree mode mstp
!
interface Ethernet1
   description [dev:DC1-SW3-LEAF1][int:ETH1]
   mtu 9000
   no switchport
   ip address 10.0.1.0/31
!
interface Ethernet2
   description [dev:DC1-SW3-LEAF2][int:ETH1]
   mtu 9000
   no switchport
   ip address 10.0.1.2/31
!
interface Ethernet3
   description [dev:DC1-SW3-LEAF3][int:ETH1]
   mtu 9000
   no switchport
   ip address 10.0.1.4/31
!
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback0
   description #UNDERLAY#
   ip address 10.0.0.1/32
!
interface Management1
   description [dev:DC1-SW-MGMT1]
   ip address 172.16.0.1/24
!
ip routing
!
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
   distance bgp 20 200 200
   maximum-paths 4 ecmp 4
   bgp listen range 10.0.1.0/24 peer-group PG_LEAF peer-filter PF_LEAF
   bgp listen range 10.0.0.0/24 peer-group PG_LEAF_EVPN peer-filter PF_LEAF
   neighbor PG_LEAF peer group
   neighbor PG_LEAF password 7 z+yWsJee8ho4Ju8e6CN9sg==
   neighbor PG_LEAF send-community standard extended large
   neighbor PG_LEAF_EVPN peer group
   neighbor PG_LEAF_EVPN next-hop-unchanged
   neighbor PG_LEAF_EVPN update-source Loopback0
   neighbor PG_LEAF_EVPN ebgp-multihop 3
   neighbor PG_LEAF_EVPN send-community extended
   !
   address-family evpn
      neighbor PG_LEAF_EVPN activate
   !
   address-family ipv4
      neighbor PG_LEAF activate
      network 10.0.0.1/32
!
end



### DC1-SW3-SPINE2
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname DC1-SW3-SPINE2
!
spanning-tree mode mstp
!
interface Ethernet1
   description [dev:DC1-SW3-LEAF1][ETH2]
   mtu 9000
   no switchport
   ip address 10.0.1.6/31
!
interface Ethernet2
   description [dev:DC1-SW3-LEAF2][ETH2]
   mtu 9000
   no switchport
   ip address 10.0.1.8/31
!
interface Ethernet3
   description [dev:DC1-SW3-LEAF3][ETH2]
   mtu 9000
   no switchport
   ip address 10.0.1.10/31
!
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback0
   description #UNDERLAY#
   ip address 10.0.0.2/32
!
interface Management1
   description [dev:DC1-SW-MGMT1]
   ip address 172.16.0.2/24
!
ip routing
!
peer-filter PF_LEAF
   10 match as-range 64513-65534 result accept
!
router bfd
   interval 100 min-rx 100 multiplier 3 default
!
router bgp 64512
   router-id 10.0.0.2
   no bgp default ipv4-unicast
   timers bgp 3 9
   distance bgp 20 200 200
   maximum-paths 4 ecmp 4
   bgp listen range 10.0.1.0/24 peer-group PG_LEAF peer-filter PF_LEAF
   bgp listen range 10.0.0.0/24 peer-group PG_LEAF_EVPN peer-filter PF_LEAF
   neighbor PG_LEAF peer group
   neighbor PG_LEAF password 7 z+yWsJee8ho4Ju8e6CN9sg==
   neighbor PG_LEAF send-community standard extended large
   neighbor PG_LEAF_EVPN peer group
   neighbor PG_LEAF_EVPN next-hop-unchanged
   neighbor PG_LEAF_EVPN update-source Loopback0
   neighbor PG_LEAF_EVPN ebgp-multihop 3
   neighbor PG_LEAF_EVPN send-community extended
   !
   address-family evpn
      neighbor PG_LEAF_EVPN activate
   !
   address-family ipv4
      neighbor PG_LEAF activate
      network 10.0.0.2/32
!
end
