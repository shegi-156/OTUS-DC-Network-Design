# DC1-SW3-SPINE1

```
! Command: show running-config
! device: DC1-SW3-SPINE1 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname DC1-SW3-SPINE1
!
spanning-tree mode mstp
!
interface Ethernet1
   description DC1-SW3-LEAF1::Eth1
   no switchport
   ip address 10.0.1.0/31
!
interface Ethernet2
   description DC1-SW3-LEAF2::Eth1
   no switchport
   ip address 10.0.1.2/31
!
interface Ethernet3
   description DC1-SW3-LEAF3::Eth1
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
   description UNDERLAY
   ip address 10.0.0.1/32
!
interface Management1
   description DC1-SW-MGMT1
   ip address 172.16.0.1/24
!
ip routing
!
end
```

# DC1-SW3-SPINE2

```
! Command: show running-config
! device: DC1-SW3-SPINE2 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
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
   description DC1-SW3-LEAF1::Eth2
   no switchport
   ip address 10.0.1.6/31
!
interface Ethernet2
   description DC1-SW3-LEAF2::Eth2
   no switchport
   ip address 10.0.1.8/31
!
interface Ethernet3
   description DC1-SW3-LEAF3::Eth2
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
   description UNDERLAY
   ip address 10.0.0.2/32
!
interface Management1
   description DC1-SW-MGMT1
   ip address 172.16.0.2/24
!
ip routing
!
end
```

# DC1-SW3-LEAF1

```
! Command: show running-config
! device: DC1-SW3-LEAF1 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
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
interface Ethernet1
   description DC1-SW3-SPINE1::Eth1
   no switchport
   ip address 10.0.1.1/31
!
interface Ethernet2
   description DC1-SW3-SPINE2::Eth1
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
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback0
   description UNDERLAY
   ip address 10.0.0.3/32
!
interface Management1
   description DC1-SW-MGMT1
   ip address 172.16.0.3/24
!
ip routing
!
end

```

# DC1-SW3-LEAF2

```
! Command: show running-config
! device: DC1-SW3-LEAF2 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
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
interface Ethernet1
   description DC1-SW3-SPINE1::Eth2
   no switchport
   ip address 10.0.1.3/31
!
interface Ethernet2
   description DC1-SW3-SPINE2::Eth2
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
!
interface Loopback0
   description UNDERLAY
   ip address 10.0.0.4/32
!
interface Management1
   description DC1-SW-MGMT1
   ip address 172.16.0.4/24
!
ip routing
!
end
```

# DC1-SW3-LEAF3

```
! Command: show running-config
! device: DC1-SW3-LEAF3 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
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
interface Ethernet1
   description DC1-SW3-SPINE1::Eth3
   no switchport
   ip address 10.0.1.5/31
!
interface Ethernet2
   description DC1-SW3-SPINE2::Eth3
   no switchport
   ip address 10.0.1.10/31
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
!
interface Loopback0
   description UNDERLAY
   ip address 10.0.0.5/32
!
interface Management1
   description DC1-SW-MGMT1
   ip address 172.16.0.5/24
!
ip routing
!
end
```