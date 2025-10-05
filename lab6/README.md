## ДЗ №5
VxLAN. L3VNI
***
### Цель:
Настроить маршрутизацию в рамках Overlay между клиентами.
#### Задачи:
1. Настроите каждого клиента в своем VNI;
2. Настроите маршрутизацию между клиентами;
3. Зафиксируете в документации - план работы, адресное пространство, схему сети, конфигурацию устройств;
***
### Исходные данные:
В качестве Underlay будет использоваться eBGP, настроенный в ДЗ №4 [lab4](https://github.com/egorvshch/DC-networks-design/tree/main/lab4),
В качестве отправной точки для данной ДЗ будут использоваться настройки Overlay на основе BGP VxLAN EVPN, выполненные в рамках ДЗ №5 [lab5](https://github.com/egorvshch/DC-networks-design/tree/main/lab5),
кратко:
- Spine-1 и Spine-2 находятся в AS65000;
- Leaf-1, Leaf-2 и Leaf-3 находятся в AS65001, AS65002 и AS65003 соответственно.
- L2-связность настроена между хостами в VLAN 10 (между Host-1 (Leaf-1) и Host-3 (Leaf-3)) и VLAN 20 (между Host-2 (Leaf-2) и Host-4 (Leaf-3))

#### Схема сети:
   
![](https://github.com/egorvshch/DC-networks-design/blob/main/lab5/schema_dc_net.JPG)
---

#### IP-адресация оборудования для Underlay сети ЦОД:

<details>
<summary> Таблица IP-адресация оборудования </summary>
   
| Device | Interface	| IP Address | Subnet Mask | description |
|:------ |:-----------|:----------:|:-------------:|:-------------:|
| Spine-1	| Lo0	| 10.0.1.0	| /32 |
| | Lo1	| 10.1.1.0	| /32 |
| | Eth1	| 10.2.1.0	| /31 | to Leaf-1 |
| | Eth2	| 10.2.1.2	| /31 | to Leaf-2 |
| | Eth3	| 10.2.1.4	| /31 | to Leaf-3 |
| Spine-2	| Lo0	| 10.0.2.0	| /32 |
| | Lo1	| 10.1.2.0	| /32 |
| | Eth1	| 10.2.2.0	| /31 | to Leaf-1 |
| | Eth2	| 10.2.2.2	| /31 | to Leaf-2 |
| | Eth3	| 10.2.2.4	| /31 | to Leaf-3 |
| Leaf-1	| Lo0	| 10.0.0.1	| /32 |
| | Lo1	| 10.1.0.1	| /32 |
| | Eth1	| 10.2.1.1	| /31 | to Spine-1 |
| | Eth2	| 10.2.2.1	|/31 | to Spine-2 |
| | Eth3	| 10.4.1.1	| /24 | to Server-1 |
| Leaf-2	| Lo0	| 10.0.0.2	| /32 |
| | Lo1	| 10.1.0.2	| /32 |
| | Eth1	| 10.2.1.3	| /31 | to Spine-1 |
| | Eth2	| 10.2.2.3	| /31 | to Spine-2 |
| | Eth3	| 10.4.2.1	| /24 | to Server-2 |
| Leaf-3	| Lo0	| 10.0.0.3	| /32 |
| | Lo1	| 10.1.0.3	| /32 |
| | Eth1	| 10.2.1.5	| /31 | to Spine-1 |
| | Eth2	| 10.2.2.5	| /31 | to Spine-2 |
| | Eth3	| 10.4.3.1	| /24 | to Server-3 |
| | Eth4	| 10.4.4.1	| /24 | to Server-4 |
| Host-1	| eth0	| 10.4.10.10	| /24 | Vlan 10 |
| Host-2	| eth0	| 10.4.20.20	| /24 | Vlan 20 |
| Host-3	| eth0	| 10.4.10.30	| /24 | Vlan 10 |
| Host-4	| eth0	| 10.4.20.40	| /24 | Vlan 20 |

</details>

---
### Результаты:

#### 1. План работ (Arista):

Для связности между различными VLAN в различных VNI будем использовать Symmetric IRB.
Для этого выполним следующие настройки:

**1.1 Настриваем L3 VNI на каждом LEAF коммутаторе**, для чего:
- Добавляем IP VRF (например VRF BLUE);
- Мапим этот VRF в VNI 10001;
- Настраиваем VRF в BGP, указав RD, RT и анонсировав connected подсети данного VRF. RD и КЕ указываем вручную, RD имеет формат IP_VTEP:ID, RT имеет формат ID:VNI;

*****Конфигурация для Leaf-1*****
```
vrf instance BLUE
!
ip routing vrf BLUE
!
int vxlan1
  vxlan vrf BLUE vni 10001
!
router bgp 65001
 vrf BLUE
    rd 10.1.0.1:1
    route-target import evpn 1:10001
    route-target export evpn 1:10001
    redistribute connected
```	
*****Конфигурация для Leaf-2*****
```
vrf instance BLUE
!
ip routing vrf BLUE
!
int vxlan1
  vxlan vrf BLUE vni 10001
!
router bgp 65002
 vrf BLUE
    rd 10.1.0.2:1
    route-target import evpn 1:10001
    route-target export evpn 1:10001
    redistribute connected
```
*****Конфигурация для Leaf-2*****
```
vrf instance BLUE
!
ip routing vrf BLUE
!
int vxlan1
  vxlan vrf BLUE vni 10001
!
router bgp 65003
 vrf BLUE
    rd 10.1.0.3:1
    route-target import evpn 1:10001
    route-target export evpn 1:10001
    redistribute connected
```
##### 1.2 Настриваем SVI на Leaf-1 и Leaf-2 и Leaf-3 для каждого из клиентов, указав виртуальные Anycast Gateway IP и MAC адреса:**

*****Конфигурация для Leaf-1*****
```
ip virtual-router mac-address 00:00:00:00:00:01
!
interface Vlan10
 vrf BLUE
   ip address virtual 10.4.10.1/24
```
*****Конфигурация для Leaf-2*****
```
ip virtual-router mac-address 00:00:00:00:00:01
!
interface Vlan20
 vrf BLUE
   ip address virtual 10.4.20.1/24
```
*****Конфигурация для Leaf-3*****
```
ip virtual-router mac-address 00:00:00:00:00:01
!
interface Vlan10
 vrf BLUE
   ip address virtual 10.4.10.1/24
!
interface Vlan20
 vrf BLUE
   ip address virtual 10.4.20.1/24
```


#### 2. Полная конфигурация коммутаторов:

<details>
<summary> Spine-1 (без изменений) </summary>
  
```
Spine-1#sh running-config
! Command: show running-config
! device: Spine-1 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname Spine-1
!
spanning-tree mode mstp
!
interface Ethernet1
   description to Leaf-1
   no switchport
   ip address 10.2.1.0/31
!
interface Ethernet2
   description to Leaf-2
   no switchport
   ip address 10.2.1.2/31
!
interface Ethernet3
   description to Leaf-3
   no switchport
   ip address 10.2.1.4/31
!
interface Ethernet4
   no switchport
!
interface Ethernet5
   no switchport
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback0
   ip address 10.0.1.0/32
!
interface Loopback1
   ip address 10.1.1.0/32
!
interface Management1
!
ip routing
!
ip prefix-list DIRECT_CONNECTED
   seq 10 permit 10.0.1.0/32
   seq 20 permit 10.1.1.0/32
!
route-map RM_REDISTRIBUTE_CONNECTED permit 10
   match ip address prefix-list DIRECT_CONNECTED
!
peer-filter EVPN
   10 match as-range 65001-65003 result accept
!
peer-filter LEAFs
   10 match as-range 65001-65003 result accept
!
router bgp 65000
   router-id 10.0.1.0
   timers bgp 3 9
   maximum-paths 2 ecmp 2
   bgp listen range 10.0.0.0/29 peer-group EVPN peer-filter EVPN
   bgp listen range 10.2.1.0/29 peer-group LEAF peer-filter LEAFs
   neighbor EVPN peer group
   neighbor EVPN next-hop-unchanged
   neighbor EVPN update-source Loopback0
   neighbor EVPN ebgp-multihop 3
   neighbor EVPN send-community extended
   neighbor LEAF peer group
   neighbor LEAF bfd
   neighbor LEAF password 7 X924dXED0EODP1A6/K/R/w==
   neighbor LEAF send-community
   redistribute connected route-map RM_REDISTRIBUTE_CONNECTED
   !
   address-family evpn
      neighbor EVPN activate
!
end
Spine-1#
```
</details>

<details>
<summary> Spine-2 (без изменений) </summary>
  
```
Spine-2#sh running-config
! Command: show running-config
! device: Spine-2 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname Spine-2
!
spanning-tree mode mstp
!
interface Ethernet1
   description to Leaf-1
   no switchport
   ip address 10.2.2.0/31
!
interface Ethernet2
   description to Leaf-2
   no switchport
   ip address 10.2.2.2/31
!
interface Ethernet3
   description to Leaf-3
   no switchport
   ip address 10.2.2.4/31
!
interface Ethernet4
   no switchport
!
interface Ethernet5
   no switchport
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback0
   ip address 10.0.2.0/32
!
interface Loopback1
   ip address 10.1.2.0/32
!
interface Management1
!
ip routing
!
ip prefix-list DIRECT_CONNECTED
   seq 10 permit 10.0.2.0/32
   seq 20 permit 10.1.2.0/32
!
route-map RM_REDISTRIBUTE_CONNECTED permit 10
   match ip address prefix-list DIRECT_CONNECTED
!
peer-filter EVPN
   10 match as-range 65001-65003 result accept
!
peer-filter LEAFs
   10 match as-range 65001-65003 result accept
!
router bgp 65000
   router-id 10.0.2.0
   timers bgp 3 9
   maximum-paths 2 ecmp 2
   bgp listen range 10.0.0.0/29 peer-group EVPN peer-filter EVPN
   bgp listen range 10.2.2.0/29 peer-group LEAF peer-filter LEAFs
   neighbor EVPN peer group
   neighbor EVPN next-hop-unchanged
   neighbor EVPN update-source Loopback0
   neighbor EVPN ebgp-multihop 3
   neighbor EVPN send-community extended
   neighbor LEAF peer group
   neighbor LEAF bfd
   neighbor LEAF password 7 X924dXED0EODP1A6/K/R/w==
   neighbor LEAF send-community
   redistribute connected route-map RM_REDISTRIBUTE_CONNECTED
   !
   address-family evpn
      neighbor EVPN activate
!
end
Spine-2#

```
</details>

<details>
<summary> Leaf-1 </summary>
  
```
Leaf-1#sh running-config
! Command: show running-config
! device: Leaf-1 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname Leaf-1
!
spanning-tree mode mstp
!
vlan 10
   name service10
!
vrf instance BLUE
!
interface Ethernet1
   description to Spine-1
   no switchport
   ip address 10.2.1.1/31
!
interface Ethernet2
   description to Spine-2
   no switchport
   ip address 10.2.2.1/31
!
interface Ethernet3
   description Host-1
   switchport trunk allowed vlan 10
   switchport mode trunk
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
   ip address 10.0.0.1/32
!
interface Loopback1
   ip address 10.1.0.1/32
!
interface Management1
!
interface Vlan10
   vrf BLUE
   ip address virtual 10.4.10.1/24
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vrf BLUE vni 10001
   vxlan learn-restrict any
!
ip virtual-router mac-address 00:00:00:00:00:01
!
ip routing
ip routing vrf BLUE
!
ip prefix-list DIRECT_CONNECTED
   seq 10 permit 10.0.0.1/32
   seq 20 permit 10.1.0.1/32
!
route-map RM_REDISTRIBUTE_CONNECTED permit 10
   match ip address prefix-list DIRECT_CONNECTED
!
router bgp 65001
   router-id 10.0.0.1
   timers bgp 3 9
   maximum-paths 2 ecmp 2
   neighbor EVPN peer group
   neighbor EVPN remote-as 65000
   neighbor EVPN update-source Loopback0
   neighbor EVPN ebgp-multihop 3
   neighbor EVPN send-community extended
   neighbor SPINE peer group
   neighbor SPINE remote-as 65000
   neighbor SPINE bfd
   neighbor SPINE password 7 sWIT5buW1FYPyTvv4vVCmQ==
   neighbor SPINE send-community
   neighbor 10.0.1.0 peer group EVPN
   neighbor 10.0.2.0 peer group EVPN
   neighbor 10.2.1.0 peer group SPINE
   neighbor 10.2.2.0 peer group SPINE
   redistribute connected route-map RM_REDISTRIBUTE_CONNECTED
   !
   vlan 10
      rd 65001:10010
      route-target both 10:10010
      redistribute learned
   !
   address-family evpn
      neighbor EVPN activate
   !
   vrf BLUE
      rd 10.1.0.1:1
      route-target import evpn 1:10001
      route-target export evpn 1:10001
      redistribute connected
!
end
Leaf-1#


```
</details>

<details>
<summary> Leaf-2 </summary>
  
```

Leaf-2#sh running-config
! Command: show running-config
! device: Leaf-2 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname Leaf-2
!
spanning-tree mode mstp
!
vlan 20
   name service20
!
vrf instance BLUE
!
interface Ethernet1
   description to Spine-1
   no switchport
   ip address 10.2.1.3/31
!
interface Ethernet2
   description to Spine-2
   no switchport
   ip address 10.2.2.3/31
!
interface Ethernet3
   description Host-2
   switchport trunk allowed vlan 20
   switchport mode trunk
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
   ip address 10.0.0.2/32
!
interface Loopback1
   ip address 10.1.0.2/32
!
interface Management1
!
interface Vlan20
   vrf BLUE
   ip address virtual 10.4.20.1/24
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 20 vni 10020
   vxlan vrf BLUE vni 10001
   vxlan learn-restrict any
!
ip virtual-router mac-address 00:00:00:00:00:01
!
ip routing
ip routing vrf BLUE
!
ip prefix-list DIRECT_CONNECTED
   seq 10 permit 10.0.0.2/32
   seq 20 permit 10.1.0.2/32
!
route-map RM_REDISTRIBUTE_CONNECTED permit 10
   match ip address prefix-list DIRECT_CONNECTED
!
router bgp 65002
   router-id 10.0.0.2
   timers bgp 3 9
   maximum-paths 2 ecmp 2
   neighbor EVPN peer group
   neighbor EVPN remote-as 65000
   neighbor EVPN update-source Loopback0
   neighbor EVPN ebgp-multihop 3
   neighbor EVPN send-community extended
   neighbor SPINE peer group
   neighbor SPINE remote-as 65000
   neighbor SPINE bfd
   neighbor SPINE password 7 sWIT5buW1FYPyTvv4vVCmQ==
   neighbor SPINE send-community
   neighbor 10.0.1.0 peer group EVPN
   neighbor 10.0.2.0 peer group EVPN
   neighbor 10.2.1.2 peer group SPINE
   neighbor 10.2.2.2 peer group SPINE
   redistribute connected route-map RM_REDISTRIBUTE_CONNECTED
   !
   vlan 20
      rd 65002:10020
      route-target both 20:10020
      redistribute learned
   !
   address-family evpn
      neighbor EVPN activate
   !
   vrf BLUE
      rd 10.1.0.2:1
      route-target import evpn 1:10001
      route-target export evpn 1:10001
      redistribute connected
!
end
Leaf-2#

```
</details>

<details>
<summary> Leaf-3 </summary>
  
```
Leaf-3#sh running-config
! Command: show running-config
! device: Leaf-3 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname Leaf-3
!
spanning-tree mode mstp
!
vlan 10
   name service10
!
vlan 20
   name service20
!
vrf instance BLUE
!
interface Ethernet1
   description to Spine-1
   no switchport
   ip address 10.2.1.5/31
!
interface Ethernet2
   description to Spine-2
   no switchport
   ip address 10.2.2.5/31
!
interface Ethernet3
   description Host-3
   switchport trunk allowed vlan 10
   switchport mode trunk
!
interface Ethernet4
   description Host-4
   switchport trunk allowed vlan 20
   switchport mode trunk
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
   ip address 10.0.0.3/32
!
interface Loopback1
   ip address 10.1.0.3/32
!
interface Management1
!
interface Vlan10
   vrf BLUE
   ip address virtual 10.4.10.1/24
!
interface Vlan20
   vrf BLUE
   ip address virtual 10.4.20.1/24
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
   vxlan vrf BLUE vni 10001
   vxlan learn-restrict any
!
ip virtual-router mac-address 00:00:00:00:00:01
!
ip routing
ip routing vrf BLUE
!
ip prefix-list DIRECT_CONNECTED
   seq 10 permit 10.0.0.3/32
   seq 20 permit 10.1.0.3/32
!
route-map RM_REDISTRIBUTE_CONNECTED permit 10
   match ip address prefix-list DIRECT_CONNECTED
!
router bgp 65003
   router-id 10.0.0.3
   timers bgp 3 9
   maximum-paths 2 ecmp 2
   neighbor EVPN peer group
   neighbor EVPN remote-as 65000
   neighbor EVPN update-source Loopback0
   neighbor EVPN ebgp-multihop 3
   neighbor EVPN send-community extended
   neighbor SPINE peer group
   neighbor SPINE remote-as 65000
   neighbor SPINE bfd
   neighbor SPINE password 7 sWIT5buW1FYPyTvv4vVCmQ==
   neighbor SPINE send-community
   neighbor 10.0.1.0 peer group EVPN
   neighbor 10.0.2.0 peer group EVPN
   neighbor 10.2.1.4 peer group SPINE
   neighbor 10.2.2.4 peer group SPINE
   redistribute connected route-map RM_REDISTRIBUTE_CONNECTED
   !
   vlan 10
      rd 65003:10010
      route-target both 10:10010
      redistribute learned
   !
   vlan 20
      rd 65003:10020
      route-target both 20:10020
      redistribute learned
   !
   address-family evpn
      neighbor EVPN activate
   !
   vrf BLUE
      rd 10.1.0.3:1
      route-target import evpn 1:10001
      route-target export evpn 1:10001
      redistribute connected
!
end
Leaf-3#

```
</details>

---
#### 3. Проверка наличия IP связанности между хостами различных VNI:

**- c Host-1 (VNI 10010)**  до Host-2 и Host-4 (VNI 10020)
  
```
Host-1#ping 10.4.20.20
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.4.20.20, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 39/221/587 ms
Host-1#ping 10.4.20.40
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.4.20.40, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 37/48/62 ms
Host-1#
```
**- c Host-2 (VNI 10020)** до Host-1 и Host-3 (VNI 10010)

```
Host-2#ping 10.4.10.10
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.4.10.10, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 36/52/77 ms
Host-2#ping 10.4.10.30
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.4.10.30, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 39/185/762 ms
Host-2#
```
##### 3.1 Проверка информации об VTEP и VNI и NVE:

```
show vxlan vtep detail
show vxlan vni
show interface vxlan1
```

<details>
<summary> на Leaf-1 </summary>

```
Leaf-1# show vxlan vtep detail
Remote VTEPS for Vxlan1:

VTEP           Learned Via         MAC Address Learning       Tunnel Type(s)
-------------- ------------------- -------------------------- --------------
10.1.0.2       control plane       control plane              unicast
10.1.0.3       control plane       control plane              flood, unicast

Total number of remote VTEPS:  2
!
Leaf-1#show vxlan vni
VNI to VLAN Mapping for Vxlan1
VNI         VLAN       Source       Interface       802.1Q Tag
----------- ---------- ------------ --------------- ----------
10010       10         static       Ethernet3       10
                                    Vxlan1          10

VNI to dynamic VLAN Mapping for Vxlan1
VNI         VLAN       VRF        Source
----------- ---------- ---------- ------------
10001       4094       BLUE       evpn
!
!
Leaf-1#show interface vxlan1
Vxlan1 is up, line protocol is up (connected)
  Hardware is Vxlan
  Source interface is Loopback1 and is active with 10.1.0.1
  Listening on UDP port 4789
  Replication/Flood Mode is headend with Flood List Source: EVPN
  Remote MAC learning via EVPN
  VNI mapping to VLANs
  Static VLAN to VNI mapping is
    [10, 10010]
  Dynamic VLAN to VNI mapping for 'evpn' is
    [4094, 10001]
  Note: All Dynamic VLANs used by VCS are internal VLANs.
        Use 'show vxlan vni' for details.
  Static VRF to VNI mapping is
   [BLUE, 10001]
  Headend replication flood vtep list is:
    10 10.1.0.3
  Shared Router MAC is 0000.0000.0000
Leaf-1#

```
</details>

<details>
<summary> на Leaf-2 </summary>

```
Leaf-2#show vxlan vtep detail
Remote VTEPS for Vxlan1:

VTEP           Learned Via         MAC Address Learning       Tunnel Type(s)
-------------- ------------------- -------------------------- --------------
10.1.0.1       control plane       control plane              unicast
10.1.0.3       control plane       control plane              unicast, flood

Total number of remote VTEPS:  2
Leaf-2#
Leaf-2#show vxlan vni
VNI to VLAN Mapping for Vxlan1
VNI         VLAN       Source       Interface       802.1Q Tag
----------- ---------- ------------ --------------- ----------
10020       20         static       Ethernet3       20
                                    Vxlan1          20

VNI to dynamic VLAN Mapping for Vxlan1
VNI         VLAN       VRF        Source
----------- ---------- ---------- ------------
10001       4094       BLUE       evpn

Leaf-2#
Leaf-2#show interface vxlan1
Vxlan1 is up, line protocol is up (connected)
  Hardware is Vxlan
  Source interface is Loopback1 and is active with 10.1.0.2
  Listening on UDP port 4789
  Replication/Flood Mode is headend with Flood List Source: EVPN
  Remote MAC learning via EVPN
  VNI mapping to VLANs
  Static VLAN to VNI mapping is
    [20, 10020]
  Dynamic VLAN to VNI mapping for 'evpn' is
    [4094, 10001]
  Note: All Dynamic VLANs used by VCS are internal VLANs.
        Use 'show vxlan vni' for details.
  Static VRF to VNI mapping is
   [BLUE, 10001]
  Headend replication flood vtep list is:
    20 10.1.0.3
  Shared Router MAC is 0000.0000.0000
Leaf-2#

```

</details>

<details>
<summary> на Leaf-3 </summary>

```
Leaf-3#show vxlan vtep detail
Remote VTEPS for Vxlan1:

VTEP           Learned Via         MAC Address Learning       Tunnel Type(s)
-------------- ------------------- -------------------------- --------------
10.1.0.1       control plane       control plane              unicast, flood
10.1.0.2       control plane       control plane              unicast, flood

Total number of remote VTEPS:  2
Leaf-3#
Leaf-3#show vxlan vni
VNI to VLAN Mapping for Vxlan1
VNI         VLAN       Source       Interface       802.1Q Tag
----------- ---------- ------------ --------------- ----------
10010       10         static       Ethernet3       10
                                    Vxlan1          10
10020       20         static       Ethernet4       20
                                    Vxlan1          20

VNI to dynamic VLAN Mapping for Vxlan1
VNI         VLAN       VRF        Source
----------- ---------- ---------- ------------
10001       4093       BLUE       evpn

Leaf-3#
Leaf-3#show interface vxlan1
Vxlan1 is up, line protocol is up (connected)
  Hardware is Vxlan
  Source interface is Loopback1 and is active with 10.1.0.3
  Listening on UDP port 4789
  Replication/Flood Mode is headend with Flood List Source: EVPN
  Remote MAC learning via EVPN
  VNI mapping to VLANs
  Static VLAN to VNI mapping is
    [10, 10010]       [20, 10020]
  Dynamic VLAN to VNI mapping for 'evpn' is
    [4093, 10001]
  Note: All Dynamic VLANs used by VCS are internal VLANs.
        Use 'show vxlan vni' for details.
  Static VRF to VNI mapping is
   [BLUE, 10001]
  Headend replication flood vtep list is:
    10 10.1.0.1
    20 10.1.0.2
  Shared Router MAC is 0000.0000.0000
Leaf-3#

```
</details>

##### 3.2 Проверка маршрутов BGP EVPN:
В выводах видно, что присутствуют маршруты Type-5 (ip-prefix), а также есть маршруты Type-2 mac-ip и mac-ip + IP:

<details>
<summary> Leaf-1#show bgp evpn </summary>

```
Leaf-1#sh bgp evpn
BGP routing table information for VRF default
Router identifier 10.0.0.1, local AS number 65001
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 65001:10010 mac-ip aabb.cc80.6000
                                 -                     -       -       0       i
 * >      RD: 65001:10010 mac-ip aabb.cc80.6000 10.4.10.10
                                 -                     -       -       0       i
 * >Ec    RD: 65002:10020 mac-ip aabb.cc80.7000
                                 10.1.0.2              -       100     0       65000 65002 i
 *  ec    RD: 65002:10020 mac-ip aabb.cc80.7000
                                 10.1.0.2              -       100     0       65000 65002 i
 * >Ec    RD: 65002:10020 mac-ip aabb.cc80.7000 10.4.20.20
                                 10.1.0.2              -       100     0       65000 65002 i
 *  ec    RD: 65002:10020 mac-ip aabb.cc80.7000 10.4.20.20
                                 10.1.0.2              -       100     0       65000 65002 i
 * >Ec    RD: 65003:10010 mac-ip aabb.cc80.8000
                                 10.1.0.3              -       100     0       65000 65003 i
 *  ec    RD: 65003:10010 mac-ip aabb.cc80.8000
                                 10.1.0.3              -       100     0       65000 65003 i
 * >Ec    RD: 65003:10010 mac-ip aabb.cc80.8000 10.4.10.30
                                 10.1.0.3              -       100     0       65000 65003 i
 *  ec    RD: 65003:10010 mac-ip aabb.cc80.8000 10.4.10.30
                                 10.1.0.3              -       100     0       65000 65003 i
 * >Ec    RD: 65003:10020 mac-ip aabb.cc80.9000
                                 10.1.0.3              -       100     0       65000 65003 i
 *  ec    RD: 65003:10020 mac-ip aabb.cc80.9000
                                 10.1.0.3              -       100     0       65000 65003 i
 * >Ec    RD: 65003:10020 mac-ip aabb.cc80.9000 10.4.20.40
                                 10.1.0.3              -       100     0       65000 65003 i
 *  ec    RD: 65003:10020 mac-ip aabb.cc80.9000 10.4.20.40
                                 10.1.0.3              -       100     0       65000 65003 i
 * >      RD: 65001:10010 imet 10.1.0.1
                                 -                     -       -       0       i
 * >Ec    RD: 65002:10020 imet 10.1.0.2
                                 10.1.0.2              -       100     0       65000 65002 i
 *  ec    RD: 65002:10020 imet 10.1.0.2
                                 10.1.0.2              -       100     0       65000 65002 i
 * >Ec    RD: 65003:10010 imet 10.1.0.3
                                 10.1.0.3              -       100     0       65000 65003 i
 *  ec    RD: 65003:10010 imet 10.1.0.3
                                 10.1.0.3              -       100     0       65000 65003 i
 * >Ec    RD: 65003:10020 imet 10.1.0.3
                                 10.1.0.3              -       100     0       65000 65003 i
 *  ec    RD: 65003:10020 imet 10.1.0.3
                                 10.1.0.3              -       100     0       65000 65003 i
 * >      RD: 10.1.0.1:1 ip-prefix 10.4.10.0/24
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.3:1 ip-prefix 10.4.10.0/24
                                 10.1.0.3              -       100     0       65000 65003 i
 *  ec    RD: 10.1.0.3:1 ip-prefix 10.4.10.0/24
                                 10.1.0.3              -       100     0       65000 65003 i
 * >Ec    RD: 10.1.0.2:1 ip-prefix 10.4.20.0/24
                                 10.1.0.2              -       100     0       65000 65002 i
 *  ec    RD: 10.1.0.2:1 ip-prefix 10.4.20.0/24
                                 10.1.0.2              -       100     0       65000 65002 i
 * >Ec    RD: 10.1.0.3:1 ip-prefix 10.4.20.0/24
                                 10.1.0.3              -       100     0       65000 65003 i
 *  ec    RD: 10.1.0.3:1 ip-prefix 10.4.20.0/24
                                 10.1.0.3              -       100     0       65000 65003 i
Leaf-1#

```
</details>

<details>
<summary> Leaf-2#show bgp evpn </summary>

```
Leaf-2#sh bgp evpn
BGP routing table information for VRF default
Router identifier 10.0.0.2, local AS number 65002
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 65001:10010 mac-ip aabb.cc80.6000
                                 10.1.0.1              -       100     0       65000 65001 i
 *  ec    RD: 65001:10010 mac-ip aabb.cc80.6000
                                 10.1.0.1              -       100     0       65000 65001 i
 * >Ec    RD: 65001:10010 mac-ip aabb.cc80.6000 10.4.10.10
                                 10.1.0.1              -       100     0       65000 65001 i
 *  ec    RD: 65001:10010 mac-ip aabb.cc80.6000 10.4.10.10
                                 10.1.0.1              -       100     0       65000 65001 i
 * >      RD: 65002:10020 mac-ip aabb.cc80.7000
                                 -                     -       -       0       i
 * >      RD: 65002:10020 mac-ip aabb.cc80.7000 10.4.20.20
                                 -                     -       -       0       i
 * >Ec    RD: 65003:10010 mac-ip aabb.cc80.8000
                                 10.1.0.3              -       100     0       65000 65003 i
 *  ec    RD: 65003:10010 mac-ip aabb.cc80.8000
                                 10.1.0.3              -       100     0       65000 65003 i
 * >Ec    RD: 65003:10010 mac-ip aabb.cc80.8000 10.4.10.30
                                 10.1.0.3              -       100     0       65000 65003 i
 *  ec    RD: 65003:10010 mac-ip aabb.cc80.8000 10.4.10.30
                                 10.1.0.3              -       100     0       65000 65003 i
 * >Ec    RD: 65003:10020 mac-ip aabb.cc80.9000
                                 10.1.0.3              -       100     0       65000 65003 i
 *  ec    RD: 65003:10020 mac-ip aabb.cc80.9000
                                 10.1.0.3              -       100     0       65000 65003 i
 * >Ec    RD: 65003:10020 mac-ip aabb.cc80.9000 10.4.20.40
                                 10.1.0.3              -       100     0       65000 65003 i
 *  ec    RD: 65003:10020 mac-ip aabb.cc80.9000 10.4.20.40
                                 10.1.0.3              -       100     0       65000 65003 i
 * >Ec    RD: 65001:10010 imet 10.1.0.1
                                 10.1.0.1              -       100     0       65000 65001 i
 *  ec    RD: 65001:10010 imet 10.1.0.1
                                 10.1.0.1              -       100     0       65000 65001 i
 * >      RD: 65002:10020 imet 10.1.0.2
                                 -                     -       -       0       i
 * >Ec    RD: 65003:10010 imet 10.1.0.3
                                 10.1.0.3              -       100     0       65000 65003 i
 *  ec    RD: 65003:10010 imet 10.1.0.3
                                 10.1.0.3              -       100     0       65000 65003 i
 * >Ec    RD: 65003:10020 imet 10.1.0.3
                                 10.1.0.3              -       100     0       65000 65003 i
 *  ec    RD: 65003:10020 imet 10.1.0.3
                                 10.1.0.3              -       100     0       65000 65003 i
 * >Ec    RD: 10.1.0.1:1 ip-prefix 10.4.10.0/24
                                 10.1.0.1              -       100     0       65000 65001 i
 *  ec    RD: 10.1.0.1:1 ip-prefix 10.4.10.0/24
                                 10.1.0.1              -       100     0       65000 65001 i
 * >Ec    RD: 10.1.0.3:1 ip-prefix 10.4.10.0/24
                                 10.1.0.3              -       100     0       65000 65003 i
 *  ec    RD: 10.1.0.3:1 ip-prefix 10.4.10.0/24
                                 10.1.0.3              -       100     0       65000 65003 i
 * >      RD: 10.1.0.2:1 ip-prefix 10.4.20.0/24
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.3:1 ip-prefix 10.4.20.0/24
                                 10.1.0.3              -       100     0       65000 65003 i
 *  ec    RD: 10.1.0.3:1 ip-prefix 10.4.20.0/24
                                 10.1.0.3              -       100     0       65000 65003 i
Leaf-2#

```
</details>

<details>
<summary> Leaf-3# show bgp evpn </summary>

```
Leaf-3#sh bgp evpn
BGP routing table information for VRF default
Router identifier 10.0.0.3, local AS number 65003
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 65001:10010 mac-ip aabb.cc80.6000
                                 10.1.0.1              -       100     0       65000 65001 i
 *  ec    RD: 65001:10010 mac-ip aabb.cc80.6000
                                 10.1.0.1              -       100     0       65000 65001 i
 * >Ec    RD: 65001:10010 mac-ip aabb.cc80.6000 10.4.10.10
                                 10.1.0.1              -       100     0       65000 65001 i
 *  ec    RD: 65001:10010 mac-ip aabb.cc80.6000 10.4.10.10
                                 10.1.0.1              -       100     0       65000 65001 i
 * >Ec    RD: 65002:10020 mac-ip aabb.cc80.7000
                                 10.1.0.2              -       100     0       65000 65002 i
 *  ec    RD: 65002:10020 mac-ip aabb.cc80.7000
                                 10.1.0.2              -       100     0       65000 65002 i
 * >Ec    RD: 65002:10020 mac-ip aabb.cc80.7000 10.4.20.20
                                 10.1.0.2              -       100     0       65000 65002 i
 *  ec    RD: 65002:10020 mac-ip aabb.cc80.7000 10.4.20.20
                                 10.1.0.2              -       100     0       65000 65002 i
 * >      RD: 65003:10010 mac-ip aabb.cc80.8000
                                 -                     -       -       0       i
 * >      RD: 65003:10010 mac-ip aabb.cc80.8000 10.4.10.30
                                 -                     -       -       0       i
 * >      RD: 65003:10020 mac-ip aabb.cc80.9000
                                 -                     -       -       0       i
 * >      RD: 65003:10020 mac-ip aabb.cc80.9000 10.4.20.40
                                 -                     -       -       0       i
 * >Ec    RD: 65001:10010 imet 10.1.0.1
                                 10.1.0.1              -       100     0       65000 65001 i
 *  ec    RD: 65001:10010 imet 10.1.0.1
                                 10.1.0.1              -       100     0       65000 65001 i
 * >Ec    RD: 65002:10020 imet 10.1.0.2
                                 10.1.0.2              -       100     0       65000 65002 i
 *  ec    RD: 65002:10020 imet 10.1.0.2
                                 10.1.0.2              -       100     0       65000 65002 i
 * >      RD: 65003:10010 imet 10.1.0.3
                                 -                     -       -       0       i
 * >      RD: 65003:10020 imet 10.1.0.3
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.1:1 ip-prefix 10.4.10.0/24
                                 10.1.0.1              -       100     0       65000 65001 i
 *  ec    RD: 10.1.0.1:1 ip-prefix 10.4.10.0/24
                                 10.1.0.1              -       100     0       65000 65001 i
 * >      RD: 10.1.0.3:1 ip-prefix 10.4.10.0/24
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.2:1 ip-prefix 10.4.20.0/24
                                 10.1.0.2              -       100     0       65000 65002 i
 *  ec    RD: 10.1.0.2:1 ip-prefix 10.4.20.0/24
                                 10.1.0.2              -       100     0       65000 65002 i
 * >      RD: 10.1.0.3:1 ip-prefix 10.4.20.0/24
                                 -                     -       -       0       i
Leaf-3#

```
</details>

##### 3.3 Проверка маршрутов в vrf BLUE:
<details>
<summary> Leaf-1# show ip route vrf BLUE </summary>

```
Leaf-1# show ip route vrf BLUE

VRF: BLUE
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

 B E      10.4.10.30/32 [200/0] via VTEP 10.1.0.3 VNI 10001 router-mac 50:00:00:15:f4:e8 local-interface Vxlan1
 C        10.4.10.0/24 is directly connected, Vlan10
 B E      10.4.20.20/32 [200/0] via VTEP 10.1.0.2 VNI 10001 router-mac 50:00:00:03:37:66 local-interface Vxlan1
 B E      10.4.20.40/32 [200/0] via VTEP 10.1.0.3 VNI 10001 router-mac 50:00:00:15:f4:e8 local-interface Vxlan1
 B E      10.4.20.0/24 [200/0] via VTEP 10.1.0.2 VNI 10001 router-mac 50:00:00:03:37:66 local-interface Vxlan1
                               via VTEP 10.1.0.3 VNI 10001 router-mac 50:00:00:15:f4:e8 local-interface Vxlan1

Leaf-1#

```
</details>

<details>
<summary> Leaf-2#show ip route vrf BLUE </summary>

```
Leaf-2#show ip route vrf BLUE

VRF: BLUE
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

 B E      10.4.10.10/32 [200/0] via VTEP 10.1.0.1 VNI 10001 router-mac 50:00:00:d5:5d:c0 local-interface Vxlan1
 B E      10.4.10.30/32 [200/0] via VTEP 10.1.0.3 VNI 10001 router-mac 50:00:00:15:f4:e8 local-interface Vxlan1
 B E      10.4.10.0/24 [200/0] via VTEP 10.1.0.1 VNI 10001 router-mac 50:00:00:d5:5d:c0 local-interface Vxlan1
                               via VTEP 10.1.0.3 VNI 10001 router-mac 50:00:00:15:f4:e8 local-interface Vxlan1
 B E      10.4.20.40/32 [200/0] via VTEP 10.1.0.3 VNI 10001 router-mac 50:00:00:15:f4:e8 local-interface Vxlan1
 C        10.4.20.0/24 is directly connected, Vlan20

Leaf-2#
```
</details>

<details>
<summary> Leaf-3#show ip route vrf BLUE </summary>

```
Leaf-3#show ip route vrf BLUE

VRF: BLUE
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

 B E      10.4.10.10/32 [200/0] via VTEP 10.1.0.1 VNI 10001 router-mac 50:00:00:d5:5d:c0 local-interface Vxlan1
 C        10.4.10.0/24 is directly connected, Vlan10
 B E      10.4.20.20/32 [200/0] via VTEP 10.1.0.2 VNI 10001 router-mac 50:00:00:03:37:66 local-interface Vxlan1
 C        10.4.20.0/24 is directly connected, Vlan20

Leaf-3#
```
</details>

##### 3.4 Проверка таблиц MAC адресов:
<details>
<summary> Leaf-1#sh vxlan address-table </summary>

```
Leaf-1#sh vxlan address-table
          Vxlan Mac Address Table
----------------------------------------------------------------------

VLAN  Mac Address     Type      Prt  VTEP             Moves   Last Move
----  -----------     ----      ---  ----             -----   ---------
  10  aabb.cc80.8000  EVPN      Vx1  10.1.0.3         1       0:00:19 ago
4094  5000.0003.3766  EVPN      Vx1  10.1.0.2         1       1:07:46 ago
4094  5000.0015.f4e8  EVPN      Vx1  10.1.0.3         1       1:05:19 ago
Total Remote Mac Addresses for this criterion: 3
Leaf-1#sh mac address-table
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
   1    0000.0000.0001    STATIC      Cpu
  10    0000.0000.0001    STATIC      Cpu
  10    aabb.cc80.6000    DYNAMIC     Et3        1       0:00:49 ago
  10    aabb.cc80.8000    DYNAMIC     Vx1        1       0:00:34 ago
4094    0000.0000.0001    STATIC      Cpu
4094    5000.0003.3766    DYNAMIC     Vx1        1       1:08:01 ago
4094    5000.0015.f4e8    DYNAMIC     Vx1        1       1:05:34 ago
Total Mac Addresses for this criterion: 7

          Multicast Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       ----        -----
Total Mac Addresses for this criterion: 0
Leaf-1#

```
</details>

<details>
<summary> Leaf-2#sh vxlan address-table </summary>

```
Leaf-2#sh vxlan address-table
          Vxlan Mac Address Table
----------------------------------------------------------------------

VLAN  Mac Address     Type      Prt  VTEP             Moves   Last Move
----  -----------     ----      ---  ----             -----   ---------
  20  aabb.cc80.9000  EVPN      Vx1  10.1.0.3         1       0:01:00 ago
4094  5000.0015.f4e8  EVPN      Vx1  10.1.0.3         1       1:05:45 ago
4094  5000.00d5.5dc0  EVPN      Vx1  10.1.0.1         1       1:08:58 ago
Total Remote Mac Addresses for this criterion: 3
Leaf-2#sh mac address-table
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
   1    0000.0000.0001    STATIC      Cpu
  20    0000.0000.0001    STATIC      Cpu
  20    aabb.cc80.7000    DYNAMIC     Et3        1       0:01:05 ago
  20    aabb.cc80.9000    DYNAMIC     Vx1        1       0:01:05 ago
4094    0000.0000.0001    STATIC      Cpu
4094    5000.0015.f4e8    DYNAMIC     Vx1        1       1:05:50 ago
4094    5000.00d5.5dc0    DYNAMIC     Vx1        1       1:09:03 ago
Total Mac Addresses for this criterion: 7

          Multicast Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       ----        -----
Total Mac Addresses for this criterion: 0
Leaf-2#

```
</details>

<details>
<summary> Leaf-3#sh vxlan address-table </summary>

```

          Vxlan Mac Address Table
----------------------------------------------------------------------

VLAN  Mac Address     Type      Prt  VTEP             Moves   Last Move
----  -----------     ----      ---  ----             -----   ---------
  10  aabb.cc80.6000  EVPN      Vx1  10.1.0.1         1       0:01:19 ago
  20  aabb.cc80.7000  EVPN      Vx1  10.1.0.2         1       0:01:19 ago
4093  5000.0003.3766  EVPN      Vx1  10.1.0.2         1       1:08:32 ago
4093  5000.00d5.5dc0  EVPN      Vx1  10.1.0.1         1       1:09:17 ago
Total Remote Mac Addresses for this criterion: 4
Leaf-3#sh mac address-table
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
   1    0000.0000.0001    STATIC      Cpu
  10    0000.0000.0001    STATIC      Cpu
  10    aabb.cc80.6000    DYNAMIC     Vx1        1       0:01:24 ago
  10    aabb.cc80.8000    DYNAMIC     Et3        1       0:01:09 ago
  20    0000.0000.0001    STATIC      Cpu
  20    aabb.cc80.7000    DYNAMIC     Vx1        1       0:01:23 ago
  20    aabb.cc80.9000    DYNAMIC     Et4        1       0:01:24 ago
4093    0000.0000.0001    STATIC      Cpu
4093    5000.0003.3766    DYNAMIC     Vx1        1       1:08:36 ago
4093    5000.00d5.5dc0    DYNAMIC     Vx1        1       1:09:22 ago
Total Mac Addresses for this criterion: 10

          Multicast Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       ----        -----
Total Mac Addresses for this criterion: 0
Leaf-3#

```
</details>

##### 3.5 Детальная проверка маршрута Type-5 до префиксов 10.4.10.0/24 и 10.4.20.0/24 на Leaf-1:

<details>
<summary> show bgp evpn route-type ip-prefix 10.4.10.0/24 detail </summary>

```
Leaf-1#show bgp evpn route-type ip-prefix 10.4.10.0/24 detail
BGP routing table information for VRF default
Router identifier 10.0.0.1, local AS number 65001
BGP routing table entry for ip-prefix 10.4.10.0/24, Route Distinguisher: 10.1.0.1:1
 Paths: 1 available
  Local
    - from - (0.0.0.0)
      Origin IGP, metric -, localpref -, weight 0, tag 0, valid, local, best, redistributed (Connected)
      Extended Community: Route-Target-AS:1:10001 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:d5:5d:c0
      VNI: 10001
BGP routing table entry for ip-prefix 10.4.10.0/24, Route Distinguisher: 10.1.0.3:1
 Paths: 2 available
  65000 65003
    10.1.0.3 from 10.0.1.0 (10.0.1.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:1:10001 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:15:f4:e8
      VNI: 10001
  65000 65003
    10.1.0.3 from 10.0.2.0 (10.0.2.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:1:10001 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:15:f4:e8
      VNI: 10001

```
</details>

<details>
<summary> show bgp evpn route-type ip-prefix 10.4.20.0/24 detail </summary>

```
Leaf-1#show bgp evpn route-type ip-prefix 10.4.20.0/24 detail
BGP routing table information for VRF default
Router identifier 10.0.0.1, local AS number 65001
BGP routing table entry for ip-prefix 10.4.20.0/24, Route Distinguisher: 10.1.0.2:1
 Paths: 2 available
  65000 65002
    10.1.0.2 from 10.0.1.0 (10.0.1.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:1:10001 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:03:37:66
      VNI: 10001
  65000 65002
    10.1.0.2 from 10.0.2.0 (10.0.2.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:1:10001 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:03:37:66
      VNI: 10001
BGP routing table entry for ip-prefix 10.4.20.0/24, Route Distinguisher: 10.1.0.3:1
 Paths: 2 available
  65000 65003
    10.1.0.3 from 10.0.1.0 (10.0.1.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:1:10001 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:15:f4:e8
      VNI: 10001
  65000 65003
    10.1.0.3 from 10.0.2.0 (10.0.2.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:1:10001 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:15:f4:e8
      VNI: 10001
Leaf-1#
```
</details>

##### 3.6 Детальная проверка маршрута Type-2 до MAC адреса HOST-2 (aabb.cc80.7000) и HOST-4 (aabb.cc80.9000) VNI 10020 на Leaf-1 (VNI 10010):

<details>
<summary> Leaf-1#sh bgp evpn route-type mac-ip aabb.cc80.7000 detail </summary>

```
Leaf-1#sh bgp evpn route-type mac-ip aabb.cc80.7000 detail
BGP routing table information for VRF default
Router identifier 10.0.0.1, local AS number 65001
BGP routing table entry for mac-ip aabb.cc80.7000, Route Distinguisher: 65002:10020
 Paths: 2 available
  65000 65002
    10.1.0.2 from 10.0.1.0 (10.0.1.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:20:10020 TunnelEncap:tunnelTypeVxlan
      VNI: 10020 ESI: 0000:0000:0000:0000:0000
  65000 65002
    10.1.0.2 from 10.0.2.0 (10.0.2.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:20:10020 TunnelEncap:tunnelTypeVxlan
      VNI: 10020 ESI: 0000:0000:0000:0000:0000
BGP routing table entry for mac-ip aabb.cc80.7000 10.4.20.20, Route Distinguisher: 65002:10020
 Paths: 2 available
  65000 65002
    10.1.0.2 from 10.0.1.0 (10.0.1.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:1:10001 Route-Target-AS:20:10020 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:03:37:66
      VNI: 10020 L3 VNI: 10001 ESI: 0000:0000:0000:0000:0000
  65000 65002
    10.1.0.2 from 10.0.2.0 (10.0.2.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:1:10001 Route-Target-AS:20:10020 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:03:37:66
      VNI: 10020 L3 VNI: 10001 ESI: 0000:0000:0000:0000:0000
```
</details>

<details>
<summary> Leaf-1#sh bgp evpn route-type mac-ip aabb.cc80.9000 detail </summary>

```
Leaf-1#sh bgp evpn route-type mac-ip aabb.cc80.9000 detail
BGP routing table information for VRF default
Router identifier 10.0.0.1, local AS number 65001
BGP routing table entry for mac-ip aabb.cc80.9000, Route Distinguisher: 65003:10020
 Paths: 2 available
  65000 65003
    10.1.0.3 from 10.0.1.0 (10.0.1.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:20:10020 TunnelEncap:tunnelTypeVxlan
      VNI: 10020 ESI: 0000:0000:0000:0000:0000
  65000 65003
    10.1.0.3 from 10.0.2.0 (10.0.2.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:20:10020 TunnelEncap:tunnelTypeVxlan
      VNI: 10020 ESI: 0000:0000:0000:0000:0000
BGP routing table entry for mac-ip aabb.cc80.9000 10.4.20.40, Route Distinguisher: 65003:10020
 Paths: 2 available
  65000 65003
    10.1.0.3 from 10.0.2.0 (10.0.2.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:1:10001 Route-Target-AS:20:10020 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:15:f4:e8
      VNI: 10020 L3 VNI: 10001 ESI: 0000:0000:0000:0000:0000
  65000 65003
    10.1.0.3 from 10.0.1.0 (10.0.1.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:1:10001 Route-Target-AS:20:10020 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:15:f4:e8
      VNI: 10020 L3 VNI: 10001 ESI: 0000:0000:0000:0000:0000
Leaf-1#
```
</details>
