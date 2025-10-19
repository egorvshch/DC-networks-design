# ДЗ №7
VXLAN. Multihoming
***
## Цель:
Настроить отказоустойчивое подключение клиентов с использованием EVPN Multihoming.
### Задачи:
1. Подключите клиентов 2-я линками к различным Leaf
2. Настроите агрегированный канал со стороны клиента
3. Настроите multihoming для работы в Overlay сети. Если используете Cisco NXOS - vPC, если иной вендор - то ESI LAG (либо MC-LAG с поддержкой VXLAN)
4. Зафиксируете в документации - план работы, адресное пространство, схему сети, конфигурацию устройств
5. Опционально - протестировать отказоустойчивость - убедиться, что связнность не теряется при отключении одного из линков
***
## Результаты выполнения:
---
В качестве исходной схемы использовалась схема из ДЗ №6 [lab6](https://github.com/egorvshch/DC-networks-design/tree/main/lab6), в которую были добавлены соответствующие Leaf-1-2 и Leaf-2-2
В рамках работы выполнена настройка multihoming для обоих вариантов: ESI LAG и MC-LAG с поддержкой VXLAN на оборудовании Arista.


### 1. Схема сети:
   
![](https://github.com/egorvshch/DC-networks-design/blob/main/lab7/schema_dc_net.JPG)
---

### 2. IP-адресация оборудования для Underlay сети ЦОД:

<details>
<summary> Таблица IP-адресация оборудования </summary>
   
| Device | Interface	| IP Address | Subnet Mask | description |
|:------ |:-----------|:----------:|:-------------:|:-------------:|
| Spine-1	| Lo0	| 10.0.1.0	| /32 |
| | Lo1	| 10.1.1.0	| /32 |
| | Eth1	| 10.2.1.0	| /31 | to Leaf-1-1 |
| | Eth2	| 10.2.1.2	| /31 | to Leaf-2-1 |
| | Eth3	| 10.2.1.4	| /31 | to Leaf-3 |
| | Eth4	| 10.2.1.10	| /31 | to Leaf-1-2 |
| | Eth5	| 10.2.1.22	| /31 | to Leaf-2-2 |
| Spine-2	| Lo0	| 10.0.2.0	| /32 |
| | Lo1	| 10.1.2.0	| /32 |
| | Eth1	| 10.2.2.0	| /31 | to Leaf-1 |
| | Eth2	| 10.2.2.2	| /31 | to Leaf-2 |
| | Eth3	| 10.2.2.4	| /31 | to Leaf-3 |
| | Eth4	| 10.2.2.10	| /31 | to Leaf-1-2 |
| | Eth5	| 10.2.2.22	| /31 | to Leaf-2-2 |
| Leaf-1-1	| Lo0	| 10.0.0.1	| /32 |
| | Lo1	| 10.1.0.1	| /32 |
| | Eth1	| 10.2.1.1	| /31 | to Spine-1 |
| | Eth2	| 10.2.2.1	|/31 | to Spine-2 |
| | Eth3	| 10.4.10.1	| /24 | to Host-1 |
| | Eth4	| 	|  | to mlag-peer|
| | Eth5	|  |  | to mlag-peer |
| Leaf-1-2	| Lo0	| 10.0.0.11	| /32 |
| | Lo1	| 10.1.0.11	| /32 |
| | Eth1	| 10.2.1.11	| /31 | to Spine-1 |
| | Eth2	| 10.2.2.11	|/31 | to Spine-2 |
| | Eth3	| 10.4.10.1	| /24 | to Host-1 |
| | Eth4	| 	|  | to mlag-peer|
| | Eth5	|  |  | to mlag-peer |
| Leaf-2	| Lo0	| 10.0.0.2	| /32 |
| | Lo1	| 10.1.0.2	| /32 |
| | Eth1	| 10.2.1.3	| /31 | to Spine-1 |
| | Eth2	| 10.2.2.3	| /31 | to Spine-2 |
| | Eth3	| 10.4.20.1	| /24 | to Host-2 |
| Leaf-2-2	| Lo0	| 10.0.0.22	| /32 |
| | Lo1	| 10.1.0.22	| /32 |
| | Eth1	| 10.2.1.23	| /31 | to Spine-1 |
| | Eth2	| 10.2.2.23	| /31 | to Spine-2 |
| | Eth3	| 10.4.20.1	| /24 | to Host-2 |
| Leaf-3	| Lo0	| 10.0.0.3	| /32 |
| | Lo1	| 10.1.0.3	| /32 |
| | Eth1	| 10.2.1.5	| /31 | to Spine-1 |
| | Eth2	| 10.2.2.5	| /31 | to Spine-2 |
| | Eth3	| 10.4.10.1	| /24 | to Host-3 |
| | Eth4	| 10.4.20.1	| /24 | to Host-4 |
| Host-1	| eth0	| 10.4.10.10	| /24 | Vlan 10 |
| Host-2	| eth0	| 10.4.20.20	| /24 | Vlan 20 |
| Host-3	| eth0	| 10.4.10.30	| /24 | Vlan 10 |
| Host-4	| eth0	| 10.4.20.40	| /24 | Vlan 20 |

</details>

---

### 3. Настройка MC-LAG на Leaf-1-1 и Leaf-1-2:

<details>
<summary> Настройка MC-LAG на Leaf-1-1 </summary>
   
```
#### Настройка mlag на Leaf1-1
!  
vlan 4090
 name mlag-peer
 trunk group mlag-peer
!  
interface Vlan4090
   description MLAG Peer-address
   ip address 10.199.250.1/30
!  
interface Port-Channel 999
   description mlag_peer-link_to_Leaf-1-2
   switchport mode trunk
   switchport trunk group mlag-peer
!  
interface Ethernet4
   description mlag_peer-link_to_Leaf-1-2_Eth4
   channel-group 999 mode active
!  
interface Ethernet5
   description mlag_peer-link_to_Leaf-1-2_Eth5
   channel-group 999 mode active
!  
!    
no spanning-tree vlan-id 4090
!  
mlag configuration
   domain-id mlag-999
   local-interface Vlan4090
   peer-address 10.199.250.2
   peer-link Port-Channel 999
   no shutdown
! 
#### Настройка интерфейса для iBGP:
!
vlan 4091
 name mlag-iBGP
 trunk group mlag-peer
!
interface Vlan4091
   description for_iBGP
   ip address 10.199.251.1/30
!   
no spanning-tree vlan-id 4091
!
router bgp 65001
   neighbor 10.199.251.2 next-hop-self
   neighbor 10.199.251.2 remote-as 65001
address-family ipv4
neighbor 10.199.251.2 activate
!

####  Настройка Port-Channel в сторону Host-1:
!
interface Port-Channel10
   description to Host-1
   switchport mode trunk
   switchport trunk allowed vlan 10
   mlag 10   
!  
interface Ethernet3
   description Host-1_eth0/0
   switchport trunk allowed vlan 10
   switchport mode trunk
   channel-group 10 mode active
!

```

</details>

<details>
<summary> Настройка MC-LAG на Leaf-1-2 </summary>
   
```
####  Настройка mlag на Leaf1-2
!  
vlan 4090
 name mlag-peer
 trunk group mlag-peer
!  
interface Vlan4090
   description MLAG Peer-address
   ip address 10.199.250.2/30
!  
interface Port-Channel 999
   description mlag_peer-link_to_Leaf-1-1
   switchport mode trunk
   switchport trunk group mlag-peer
!  
interface Ethernet4
   description mlag_peer-link_to_Leaf-1-1_Eth4
   channel-group 999 mode active
!  
interface Ethernet5
   description mlag_peer-link_to_Leaf-1-1_Eth5
   channel-group 999 mode active
!  
!    
no spanning-tree vlan-id 4090
!  
mlag configuration
   domain-id mlag-999
   local-interface Vlan4090
   peer-address 10.199.250.1
   peer-link Port-Channel 999
   no shutdown
! 

####  Настройка интерфейса для iBGP:
!
vlan 4091
 name mlag-iBGP
 trunk group mlag-peer
!
interface Vlan4091
   description for_iBGP
   ip address 10.199.251.2/30
!   
no spanning-tree vlan-id 4091
!
router bgp 65001
   neighbor 10.199.251.1 next-hop-self
   neighbor 10.199.251.1 remote-as 65001
address-family ipv4
neighbor 10.199.251.1 activate
!

####  Настройка Port-Channel в сторону Host-1:
!
interface Port-Channel10
   description to Host-1
   switchport mode trunk
   switchport trunk allowed vlan 10
   mlag 10   
!  
interface Ethernet3
   description Host-1_eth0/1
   switchport trunk allowed vlan 10
   switchport mode trunk
   channel-group 10 mode active
!
```

</details>



### 4. Настройка ESI LAG на Leaf-2-1 и Leaf-2-2 LEAF:

<details>
<summary> Найстройка ESI LAG на Leaf-2-1 и Leaf-2-2   </summary>
   
```
#### Найстройка ESI LAG на Leaf-2-1 и Leaf-2-2 

 interface Port-Channel20
     description to Host-2
     switchport mode trunk
	 switchport trunk allowed vlan 20
     !
     evpn ethernet-segment
        identifier 0000:0000:0000:2122:0020
        designated-forwarder election algorithm preference 100 ### (90 на Leaf-2-2)
        route-target import 00:00:21:22:00:20
     lacp system-id 0000.0021.0022

#### 	 Настройка Port-Channel в сторону Host-2:
interface Ethernet3
   description Host-2
   switchport trunk allowed vlan 20
   switchport mode trunk
   channel-group 20 mode active


```

</details>

### 5. Полная конфигурация оборудования:

[Настройка Spine-1](https://github.com/egorvshch/DC-networks-design/blob/main/lab7/configs/Spine-1)

[Настройка Spine-2](https://github.com/egorvshch/DC-networks-design/blob/main/lab7/configs/Spine-2)

[Настройка Leaf-1-1](https://github.com/egorvshch/DC-networks-design/blob/main/lab7/configs/Leaf-1-1)

[Настройка Leaf-1-2](https://github.com/egorvshch/DC-networks-design/blob/main/lab7/configs/Leaf-1-2)

[Настройка Leaf-2-1](https://github.com/egorvshch/DC-networks-design/blob/main/lab7/configs/Leaf-2-1)

[Настройка Leaf-2-2](https://github.com/egorvshch/DC-networks-design/blob/main/lab7/configs/Leaf-2-2)

[Настройка Leaf-3](https://github.com/egorvshch/DC-networks-design/blob/main/lab7/configs/Leaf-3)

[Настройка Host-1](https://github.com/egorvshch/DC-networks-design/blob/main/lab7/configs/Host-1)

[Настройка Host-2](https://github.com/egorvshch/DC-networks-design/blob/main/lab7/configs/Host-2)

[Настройка Host-3](https://github.com/egorvshch/DC-networks-design/blob/main/lab7/configs/Host-3)

[Настройка Host-4](https://github.com/egorvshch/DC-networks-design/blob/main/lab7/configs/Host-4)

### 5. Проверка наличия IP связанности между хостами:

<details>
<summary> с Host-1 </summary>
  
```
Host-1#ping 10.4.10.30
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.4.10.30, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 124/242/415 ms
Host-1#ping 10.4.20.40
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.4.20.40, timeout is 2 seconds:
.!!!!
Success rate is 80 percent (4/5), round-trip min/avg/max = 50/207/666 ms
Host-1#ping 10.4.20.20
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.4.20.20, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 41/52/71 ms
Host-1#
Host-1#sh lacp neighbor
Flags:  S - Device is requesting Slow LACPDUs
        F - Device is requesting Fast LACPDUs
        A - Device is in Active mode       P - Device is in Passive mode

Channel group 10 neighbors

Partner's information:

                  LACP port                        Admin  Oper   Port    Port
Port      Flags   Priority  Dev ID          Age    key    Key    Number  State
Et0/0     SA      32768     5200.0088.fe27  13s    0x0    0xA    0x8003  0x3D
Et0/1     SA      32768     5200.0088.fe27  25s    0x0    0xA    0x3     0x3D
Host-1#

```
</details>

<details>
<summary> с Host-2 </summary>
  
```
Host-2#ping 10.4.10.10
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.4.10.10, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 38/49/61 ms
Host-2#ping 10.4.10.30
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.4.10.30, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 37/63/146 ms
Host-2#ping 10.4.20.40
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.4.20.40, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 124/179/253 ms
Host-2#
Host-2#sh lacp neighbor
Flags:  S - Device is requesting Slow LACPDUs
        F - Device is requesting Fast LACPDUs
        A - Device is in Active mode       P - Device is in Passive mode

Channel group 20 neighbors

Partner's information:

                  LACP port                        Admin  Oper   Port    Port
Port      Flags   Priority  Dev ID          Age    key    Key    Number  State
Et0/0     SA      32768     0000.0021.0022  28s    0x0    0x14   0x3     0x3D
Et0/1     SA      32768     0000.0021.0022  21s    0x0    0x14   0x3     0x3D
Host-2#
```
</details>

### 5. Проверка настроек на коммутаторах:

<details>
<summary> на Spine-1 </summary>
  
```
Spine-1#sh bgp summary
BGP summary information for VRF default
Router identifier 10.0.1.0, local AS number 65000
Neighbor           AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc
--------- ----------- ------------- ----------------------- -------------- ---------- ----------
10.0.0.1        65001 Established   IPv4 Unicast            Negotiated              4          4
10.0.0.1        65001 Established   L2VPN EVPN              Negotiated              2          2
10.0.0.2        65002 Established   IPv4 Unicast            Negotiated              2          2
10.0.0.2        65002 Established   L2VPN EVPN              Negotiated              7          7
10.0.0.3        65003 Established   IPv4 Unicast            Negotiated              2          2
10.0.0.3        65003 Established   L2VPN EVPN              Negotiated              4          4
10.0.0.11       65001 Established   IPv4 Unicast            Negotiated              4          4
10.0.0.11       65001 Established   L2VPN EVPN              Negotiated              2          2
10.0.0.22       65022 Established   L2VPN EVPN              Negotiated              5          5
10.2.1.1        65001 Established   IPv4 Unicast            Negotiated              4          4
10.2.1.3        65002 Established   IPv4 Unicast            Negotiated              2          2
10.2.1.5        65003 Established   IPv4 Unicast            Negotiated              2          2
10.2.1.11       65001 Established   IPv4 Unicast            Negotiated              4          4
10.2.1.23       65022 Established   IPv4 Unicast            Negotiated              2          2
Spine-1#
Spine-1#sh bgp evpn
BGP routing table information for VRF default
Router identifier 10.0.1.0, local AS number 65000
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 65002:10020 auto-discovery 0 0000:0000:0000:2122:0020
                                 10.1.0.2              -       100     0       65002 i
 *  ec    RD: 65002:10020 auto-discovery 0 0000:0000:0000:2122:0020
                                 10.1.0.22             -       100     0       65022 i
 * >      RD: 10.1.0.2:1 auto-discovery 0000:0000:0000:2122:0020
                                 10.1.0.2              -       100     0       65002 i
 * >      RD: 10.1.0.22:1 auto-discovery 0000:0000:0000:2122:0020
                                 10.1.0.22             -       100     0       65022 i
 * >      RD: 65002:10020 mac-ip aabb.cc80.7000
                                 10.1.0.2              -       100     0       65002 i
 * >      RD: 65002:10020 mac-ip aabb.cc80.7000 10.4.20.20
                                 10.1.0.2              -       100     0       65002 i
 * >      RD: 65001:10010 imet 10.1.0.1
                                 10.1.0.1              -       100     0       65001 i
 * >      RD: 65002:10020 imet 10.1.0.2
                                 10.1.0.2              -       100     0       65002 i
 * >      RD: 65003:10010 imet 10.1.0.3
                                 10.1.0.3              -       100     0       65003 i
 * >      RD: 65003:10020 imet 10.1.0.3
                                 10.1.0.3              -       100     0       65003 i
 * >      RD: 65001:10010 imet 10.1.0.11
                                 10.1.0.11             -       100     0       65001 i
 * >      RD: 65002:10020 imet 10.1.0.22
                                 10.1.0.22             -       100     0       65022 i
 * >      RD: 10.1.0.2:1 ethernet-segment 0000:0000:0000:2122:0020 10.1.0.2
                                 10.1.0.2              -       100     0       65002 i
 * >      RD: 10.1.0.22:1 ethernet-segment 0000:0000:0000:2122:0020 10.1.0.22
                                 10.1.0.22             -       100     0       65022 i
 * >      RD: 10.1.0.1:1 ip-prefix 10.4.10.0/24
                                 10.1.0.1              -       100     0       65001 i
 * >      RD: 10.1.0.3:1 ip-prefix 10.4.10.0/24
                                 10.1.0.3              -       100     0       65003 i
 * >      RD: 10.1.0.11:1 ip-prefix 10.4.10.0/24
                                 10.1.0.11             -       100     0       65001 i
 * >      RD: 10.1.0.2:1 ip-prefix 10.4.20.0/24
                                 10.1.0.2              -       100     0       65002 i
 * >      RD: 10.1.0.3:1 ip-prefix 10.4.20.0/24
                                 10.1.0.3              -       100     0       65003 i
 * >      RD: 10.1.0.22:1 ip-prefix 10.4.20.0/24
5022 i
Spine-1#

```
</details>

<details>
<summary> на Spine-2 </summary>
  
```
Spine-2#   sh bgp summary
BGP summary information for VRF default
Router identifier 10.0.2.0, local AS number 65000
Neighbor           AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc
--------- ----------- ------------- ----------------------- -------------- ---------- ----------
10.0.0.1        65001 Established   IPv4 Unicast            Negotiated              4          4
10.0.0.1        65001 Established   L2VPN EVPN              Negotiated              2          2
10.0.0.2        65002 Established   IPv4 Unicast            Negotiated              2          2
10.0.0.2        65002 Established   L2VPN EVPN              Negotiated              7          7
10.0.0.3        65003 Established   IPv4 Unicast            Negotiated              2          2
10.0.0.3        65003 Established   L2VPN EVPN              Negotiated              4          4
10.0.0.11       65001 Established   IPv4 Unicast            Negotiated              4          4
10.0.0.11       65001 Established   L2VPN EVPN              Negotiated              2          2
10.0.0.22       65022 Established   L2VPN EVPN              Negotiated              5          5
10.2.2.1        65001 Established   IPv4 Unicast            Negotiated              4          4
10.2.2.3        65002 Established   IPv4 Unicast            Negotiated              2          2
10.2.2.5        65003 Established   IPv4 Unicast            Negotiated              2          2
10.2.2.11       65001 Established   IPv4 Unicast            Negotiated              4          4
10.2.2.23       65022 Established   IPv4 Unicast            Negotiated              2          2
Spine-2#
Spine-2#   sh bgp evpn
BGP routing table information for VRF default
Router identifier 10.0.2.0, local AS number 65000
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 65002:10020 auto-discovery 0 0000:0000:0000:2122:0020
                                 10.1.0.2              -       100     0       65002 i
 *  ec    RD: 65002:10020 auto-discovery 0 0000:0000:0000:2122:0020
                                 10.1.0.22             -       100     0       65022 i
 * >      RD: 10.1.0.2:1 auto-discovery 0000:0000:0000:2122:0020
                                 10.1.0.2              -       100     0       65002 i
 * >      RD: 10.1.0.22:1 auto-discovery 0000:0000:0000:2122:0020
                                 10.1.0.22             -       100     0       65022 i
 * >      RD: 65002:10020 mac-ip aabb.cc80.7000
                                 10.1.0.2              -       100     0       65002 i
 * >      RD: 65002:10020 mac-ip aabb.cc80.7000 10.4.20.20
                                 10.1.0.2              -       100     0       65002 i
 * >      RD: 65001:10010 imet 10.1.0.1
                                 10.1.0.1              -       100     0       65001 i
 * >      RD: 65002:10020 imet 10.1.0.2
                                 10.1.0.2              -       100     0       65002 i
 * >      RD: 65003:10010 imet 10.1.0.3
                                 10.1.0.3              -       100     0       65003 i
 * >      RD: 65003:10020 imet 10.1.0.3
                                 10.1.0.3              -       100     0       65003 i
 * >      RD: 65001:10010 imet 10.1.0.11
                                 10.1.0.11             -       100     0       65001 i
 * >      RD: 65002:10020 imet 10.1.0.22
                                 10.1.0.22             -       100     0       65022 i
 * >      RD: 10.1.0.2:1 ethernet-segment 0000:0000:0000:2122:0020 10.1.0.2
                                 10.1.0.2              -       100     0       65002 i
 * >      RD: 10.1.0.22:1 ethernet-segment 0000:0000:0000:2122:0020 10.1.0.22
                                 10.1.0.22             -       100     0       65022 i
 * >      RD: 10.1.0.1:1 ip-prefix 10.4.10.0/24
                                 10.1.0.1              -       100     0       65001 i
 * >      RD: 10.1.0.3:1 ip-prefix 10.4.10.0/24
                                 10.1.0.3              -       100     0       65003 i
 * >      RD: 10.1.0.11:1 ip-prefix 10.4.10.0/24
                                 10.1.0.11             -       100     0       65001 i
 * >      RD: 10.1.0.2:1 ip-prefix 10.4.20.0/24
                                 10.1.0.2              -       100     0       65002 i
 * >      RD: 10.1.0.3:1 ip-prefix 10.4.20.0/24
                                 10.1.0.3              -       100     0       65003 i
 * >      RD: 10.1.0.22:1 ip-prefix 10.4.20.0/24
5022 i
Spine-2#

```
</details>

<details>
<summary> на Leaf-1-1 </summary>
  
```
Leaf-1-1#sh bgp summary
BGP summary information for VRF default
Router identifier 10.0.0.1, local AS number 65001
Neighbor              AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc
------------ ----------- ------------- ----------------------- -------------- ---------- ----------
10.0.1.0           65000 Established   IPv4 Unicast            Negotiated              8          8
10.0.1.0           65000 Established   L2VPN EVPN              Negotiated             15         15
10.0.2.0           65000 Established   IPv4 Unicast            Negotiated              8          8
10.0.2.0           65000 Established   L2VPN EVPN              Negotiated             15         15
10.2.1.0           65000 Established   IPv4 Unicast            Negotiated              8          8
10.2.2.0           65000 Established   IPv4 Unicast            Negotiated              8          8
10.199.251.2       65001 Established   IPv4 Unicast            Negotiated             12         12
Leaf-1-1#
Leaf-1-1#show ip bgp
BGP routing table information for VRF default
Router identifier 10.0.0.1, local AS number 65001
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.0.0.1/32            -                     -       -          -       0       i
 * >Ec    10.0.0.2/32            10.2.1.0              0       -          100     0       65000 65002 i
 *  ec    10.0.0.2/32            10.2.2.0              0       -          100     0       65000 65002 i
 *        10.0.0.2/32            10.199.251.2          0       -          100     0       65000 65002 i
          10.0.0.2/32            10.2.2.3              0       -          100     0       65000 65002 i
          10.0.0.2/32            10.2.1.3              0       -          100     0       65000 65002 i
 * >Ec    10.0.0.3/32            10.2.1.0              0       -          100     0       65000 65003 i
 *  ec    10.0.0.3/32            10.2.2.0              0       -          100     0       65000 65003 i
 *        10.0.0.3/32            10.199.251.2          0       -          100     0       65000 65003 i
          10.0.0.3/32            10.2.2.5              0       -          100     0       65000 65003 i
          10.0.0.3/32            10.2.1.5              0       -          100     0       65000 65003 i
 * >      10.0.0.11/32           10.199.251.2          0       -          100     0       i
 * >Ec    10.0.0.22/32           10.2.1.0              0       -          100     0       65000 65022 i
 *  ec    10.0.0.22/32           10.2.2.0              0       -          100     0       65000 65022 i
 *        10.0.0.22/32           10.199.251.2          0       -          100     0       65000 65022 i
          10.0.0.22/32           10.2.1.23             0       -          100     0       65000 65022 i
          10.0.0.22/32           10.2.2.23             0       -          100     0       65000 65022 i
 * >      10.0.1.0/32            10.2.1.0              0       -          100     0       65000 i
 *        10.0.1.0/32            10.0.1.0              0       -          100     0       65000 i
 *        10.0.1.0/32            10.199.251.2          0       -          100     0       65000 i
 * >      10.0.2.0/32            10.2.2.0              0       -          100     0       65000 i
 *        10.0.2.0/32            10.0.2.0              0       -          100     0       65000 i
 *        10.0.2.0/32            10.199.251.2          0       -          100     0       65000 i
 * >      10.1.0.1/32            -                     -       -          -       0       i
 * >Ec    10.1.0.2/32            10.2.1.0              0       -          100     0       65000 65002 i
 *  ec    10.1.0.2/32            10.2.2.0              0       -          100     0       65000 65002 i
 *        10.1.0.2/32            10.199.251.2          0       -          100     0       65000 65002 i
          10.1.0.2/32            10.2.2.3              0       -          100     0       65000 65002 i
          10.1.0.2/32            10.2.1.3              0       -          100     0       65000 65002 i
 * >Ec    10.1.0.3/32            10.2.1.0              0       -          100     0       65000 65003 i
 *  ec    10.1.0.3/32            10.2.2.0              0       -          100     0       65000 65003 i
 *        10.1.0.3/32            10.199.251.2          0       -          100     0       65000 65003 i
          10.1.0.3/32            10.2.2.5              0       -          100     0       65000 65003 i
          10.1.0.3/32            10.2.1.5              0       -          100     0       65000 65003 i
 * >      10.1.0.11/32           10.199.251.2          0       -          100     0       i
 * >Ec    10.1.0.22/32           10.2.1.0              0       -          100     0       65000 65022 i
 *  ec    10.1.0.22/32           10.2.2.0              0       -          100     0       65000 65022 i
 *        10.1.0.22/32           10.199.251.2          0       -          100     0       65000 65022 i
          10.1.0.22/32           10.2.1.23             0       -          100     0       65000 65022 i
          10.1.0.22/32           10.2.2.23             0       -          100     0       65000 65022 i
 * >      10.1.1.0/32            10.2.1.0              0       -          100     0       65000 i
 *        10.1.1.0/32            10.0.1.0              0       -          100     0       65000 i
 *        10.1.1.0/32            10.199.251.2          0       -          100     0       65000 i
 * >      10.1.2.0/32            10.2.2.0              0       -          100     0       65000 i
 *        10.1.2.0/32            10.0.2.0              0       -          100     0       65000 i
 *        10.1.2.0/32            10.199.251.2          0       -          100     0       65000 i
Leaf-1-1#
Leaf-1-1#
Leaf-1-1#show bgp evpn
BGP routing table information for VRF default
Router identifier 10.0.0.1, local AS number 65001
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 65002:10020 auto-discovery 0 0000:0000:0000:2122:0020
                                 10.1.0.2              -       100     0       65000 65002 i
 *  ec    RD: 65002:10020 auto-discovery 0 0000:0000:0000:2122:0020
                                 10.1.0.2              -       100     0       65000 65002 i
 * >Ec    RD: 10.1.0.2:1 auto-discovery 0000:0000:0000:2122:0020
                                 10.1.0.2              -       100     0       65000 65002 i
 *  ec    RD: 10.1.0.2:1 auto-discovery 0000:0000:0000:2122:0020
                                 10.1.0.2              -       100     0       65000 65002 i
 * >Ec    RD: 10.1.0.22:1 auto-discovery 0000:0000:0000:2122:0020
                                 10.1.0.22             -       100     0       65000 65022 i
 *  ec    RD: 10.1.0.22:1 auto-discovery 0000:0000:0000:2122:0020
                                 10.1.0.22             -       100     0       65000 65022 i
 * >Ec    RD: 65002:10020 mac-ip aabb.cc80.7000
                                 10.1.0.2              -       100     0       65000 65002 i
 *  ec    RD: 65002:10020 mac-ip aabb.cc80.7000
                                 10.1.0.2              -       100     0       65000 65002 i
 * >Ec    RD: 65002:10020 mac-ip aabb.cc80.7000 10.4.20.20
                                 10.1.0.2              -       100     0       65000 65002 i
 *  ec    RD: 65002:10020 mac-ip aabb.cc80.7000 10.4.20.20
                                 10.1.0.2              -       100     0       65000 65002 i
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
 * >Ec    RD: 65002:10020 imet 10.1.0.22
                                 10.1.0.22             -       100     0       65000 65022 i
 *  ec    RD: 65002:10020 imet 10.1.0.22
                                 10.1.0.22             -       100     0       65000 65022 i
 * >Ec    RD: 10.1.0.2:1 ethernet-segment 0000:0000:0000:2122:0020 10.1.0.2
                                 10.1.0.2              -       100     0       65000 65002 i
 *  ec    RD: 10.1.0.2:1 ethernet-segment 0000:0000:0000:2122:0020 10.1.0.2
                                 10.1.0.2              -       100     0       65000 65002 i
 * >Ec    RD: 10.1.0.22:1 ethernet-segment 0000:0000:0000:2122:0020 10.1.0.22
                                 10.1.0.22             -       100     0       65000 65022 i
 *  ec    RD: 10.1.0.22:1 ethernet-segment 0000:0000:0000:2122:0020 10.1.0.22
                                 10.1.0.22             -       100     0       65000 65022 i
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
 * >Ec    RD: 10.1.0.22:1 ip-prefix 10.4.20.0/24
                                 10.1.0.22             -       100     0       65000 65022 i
 *  ec    RD: 10.1.0.22:1 ip-prefix 10.4.20.0/24
                                 10.1.0.22             -       100     0       65000 65022 i
Leaf-1-1#
Leaf-1-1#
Leaf-1-1#show ip route vrf BLUE

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


Leaf-1-1#
Leaf-1-1#show mac address-table
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
  10    0000.0000.0001    STATIC      Cpu
  10    5000.0088.fe27    STATIC      Po999
  10    aabb.cc80.6000    DYNAMIC     Po10       1       0:05:21 ago
  10    aabb.cc80.8000    DYNAMIC     Vx1        1       0:05:16 ago
4090    0000.0000.0001    STATIC      Po999
4090    5000.0088.fe27    STATIC      Po999
4091    0000.0000.0001    STATIC      Po999
4091    5000.0088.fe27    STATIC      Po999
4094    0000.0000.0001    STATIC      Po999
Total Mac Addresses for this criterion: 9

          Multicast Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       ----        -----
Total Mac Addresses for this criterion: 0
Leaf-1-1#
Leaf-1-1#
Leaf-1-1#show vxlan address-table
          Vxlan Mac Address Table
----------------------------------------------------------------------

VLAN  Mac Address     Type      Prt  VTEP             Moves   Last Move
----  -----------     ----      ---  ----             -----   ---------
  10  aabb.cc80.8000  EVPN      Vx1  10.1.0.3         1       0:06:29 ago
Total Remote Mac Addresses for this criterion: 1
Leaf-1-1#
Leaf-1-1#
Leaf-1-1#
Leaf-1-1#show mlag
MLAG Configuration:
domain-id                          :            mlag-999
local-interface                    :            Vlan4090
peer-address                       :        10.199.250.2
peer-link                          :     Port-Channel999
peer-config                        :          consistent

MLAG Status:
state                              :              Active
negotiation status                 :           Connected
peer-link status                   :                  Up
local-int status                   :                  Up
system-id                          :   52:00:00:88:fe:27
dual-primary detection             :            Disabled
dual-primary interface errdisabled :               False

MLAG Ports:
Disabled                           :                   0
Configured                         :                   0
Inactive                           :                   0
Active-partial                     :                   0
Active-full                        :                   1

Leaf-1-1#
Leaf-1-1#
Leaf-1-1#
Leaf-1-1#
Leaf-1-1#show mlag interfaces 10
                                                                   local/remote
  mlag      desc                 state       local       remote          status
--------- -------------- ---------------- ----------- ------------ ------------
    10      to Host-1      active-full        Po10         Po10           up/up
Leaf-1-1#
Leaf-1-1#
Leaf-1-1#show lacp peer
State: A = Active, P = Passive; S=ShortTimeout, L=LongTimeout;
       G = Aggregable, I = Individual; s+=InSync, s-=OutOfSync;
       C = Collecting, X = state machine expired,
       D = Distributing, d = default neighbor state
                 |                        Partner
 Port    Status  | Sys-id                    Port#   State     OperKey  PortPri
------ ----------|------------------------- ------- --------- --------- -------
Port Channel Port-Channel10*:
 Et3     Bundled | 8000,aa-bb-cc-80-60-00        1   ALGs+CD    0x000a    32768
Port Channel Port-Channel999:
 Et4     Bundled | 8000,50-00-00-88-fe-27        4   ALGs+CD    0x03e7    32768
 Et5     Bundled | 8000,50-00-00-88-fe-27        5   ALGs+CD    0x03e7    32768

* - Only local interfaces for MLAGs are displayed. Connect to the peer to
    see the state for peer interfaces.
Leaf-1-1#

```
</details>

<details>
<summary> на Leaf-1-2 </summary>
  
```
Leaf-1-2#show bgp summary
BGP summary information for VRF default
Router identifier 10.0.0.11, local AS number 65001
Neighbor              AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc
------------ ----------- ------------- ----------------------- -------------- ---------- ----------
10.0.1.0           65000 Established   IPv4 Unicast            Negotiated              8          8
10.0.1.0           65000 Established   L2VPN EVPN              Negotiated             19         19
10.0.2.0           65000 Established   IPv4 Unicast            Negotiated              8          8
10.0.2.0           65000 Established   L2VPN EVPN              Negotiated             19         19
10.2.1.10          65000 Established   IPv4 Unicast            Negotiated              8          8
10.2.2.10          65000 Established   IPv4 Unicast            Negotiated              8          8
10.199.251.1       65001 Established   IPv4 Unicast            Negotiated             12         12
Leaf-1-2#
Leaf-1-2#show ip bgp
BGP routing table information for VRF default
Router identifier 10.0.0.11, local AS number 65001
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.0.0.1/32            10.199.251.1          0       -          100     0       i
 * >Ec    10.0.0.2/32            10.2.2.10             0       -          100     0       65000 65002 i
 *  ec    10.0.0.2/32            10.2.1.10             0       -          100     0       65000 65002 i
 *        10.0.0.2/32            10.199.251.1          0       -          100     0       65000 65002 i
          10.0.0.2/32            10.2.2.3              0       -          100     0       65000 65002 i
          10.0.0.2/32            10.2.1.3              0       -          100     0       65000 65002 i
 * >Ec    10.0.0.3/32            10.2.2.10             0       -          100     0       65000 65003 i
 *  ec    10.0.0.3/32            10.2.1.10             0       -          100     0       65000 65003 i
 *        10.0.0.3/32            10.199.251.1          0       -          100     0       65000 65003 i
          10.0.0.3/32            10.2.2.5              0       -          100     0       65000 65003 i
          10.0.0.3/32            10.2.1.5              0       -          100     0       65000 65003 i
 * >      10.0.0.11/32           -                     -       -          -       0       i
 * >Ec    10.0.0.22/32           10.2.2.10             0       -          100     0       65000 65022 i
 *  ec    10.0.0.22/32           10.2.1.10             0       -          100     0       65000 65022 i
 *        10.0.0.22/32           10.199.251.1          0       -          100     0       65000 65022 i
          10.0.0.22/32           10.2.2.23             0       -          100     0       65000 65022 i
          10.0.0.22/32           10.2.1.23             0       -          100     0       65000 65022 i
 * >      10.0.1.0/32            10.2.1.10             0       -          100     0       65000 i
 *        10.0.1.0/32            10.199.251.1          0       -          100     0       65000 i
          10.0.1.0/32            10.0.1.0              0       -          100     0       65000 i
 * >      10.0.2.0/32            10.2.2.10             0       -          100     0       65000 i
 *        10.0.2.0/32            10.199.251.1          0       -          100     0       65000 i
          10.0.2.0/32            10.0.2.0              0       -          100     0       65000 i
 * >      10.1.0.1/32            10.199.251.1          0       -          100     0       i
 * >Ec    10.1.0.2/32            10.2.2.10             0       -          100     0       65000 65002 i
 *  ec    10.1.0.2/32            10.2.1.10             0       -          100     0       65000 65002 i
 *        10.1.0.2/32            10.199.251.1          0       -          100     0       65000 65002 i
          10.1.0.2/32            10.2.2.3              0       -          100     0       65000 65002 i
          10.1.0.2/32            10.2.1.3              0       -          100     0       65000 65002 i
 * >Ec    10.1.0.3/32            10.2.2.10             0       -          100     0       65000 65003 i
 *  ec    10.1.0.3/32            10.2.1.10             0       -          100     0       65000 65003 i
 *        10.1.0.3/32            10.199.251.1          0       -          100     0       65000 65003 i
          10.1.0.3/32            10.2.2.5              0       -          100     0       65000 65003 i
          10.1.0.3/32            10.2.1.5              0       -          100     0       65000 65003 i
 * >      10.1.0.11/32           -                     -       -          -       0       i
 * >Ec    10.1.0.22/32           10.2.2.10             0       -          100     0       65000 65022 i
 *  ec    10.1.0.22/32           10.2.1.10             0       -          100     0       65000 65022 i
 *        10.1.0.22/32           10.199.251.1          0       -          100     0       65000 65022 i
          10.1.0.22/32           10.2.2.23             0       -          100     0       65000 65022 i
          10.1.0.22/32           10.2.1.23             0       -          100     0       65000 65022 i
 * >      10.1.1.0/32            10.2.1.10             0       -          100     0       65000 i
 *        10.1.1.0/32            10.0.1.0              0       -          100     0       65000 i
 *        10.1.1.0/32            10.199.251.1          0       -          100     0       65000 i
 * >      10.1.2.0/32            10.2.2.10             0       -          100     0       65000 i
 *        10.1.2.0/32            10.0.2.0              0       -          100     0       65000 i
 *        10.1.2.0/32            10.199.251.1          0       -          100     0       65000 i
Leaf-1-2#  show bgp evpn
BGP routing table information for VRF default
Router identifier 10.0.0.11, local AS number 65001
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 65002:10020 auto-discovery 0 0000:0000:0000:2122:0020
                                 10.1.0.2              -       100     0       65000 65002 i
 *  ec    RD: 65002:10020 auto-discovery 0 0000:0000:0000:2122:0020
                                 10.1.0.2              -       100     0       65000 65002 i
 * >Ec    RD: 10.1.0.2:1 auto-discovery 0000:0000:0000:2122:0020
                                 10.1.0.2              -       100     0       65000 65002 i
 *  ec    RD: 10.1.0.2:1 auto-discovery 0000:0000:0000:2122:0020
                                 10.1.0.2              -       100     0       65000 65002 i
 * >Ec    RD: 10.1.0.22:1 auto-discovery 0000:0000:0000:2122:0020
                                 10.1.0.22             -       100     0       65000 65022 i
 *  ec    RD: 10.1.0.22:1 auto-discovery 0000:0000:0000:2122:0020
                                 10.1.0.22             -       100     0       65000 65022 i
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
 * >      RD: 65001:10010 imet 10.1.0.11
                                 -                     -       -       0       i
 * >Ec    RD: 65002:10020 imet 10.1.0.22
                                 10.1.0.22             -       100     0       65000 65022 i
 *  ec    RD: 65002:10020 imet 10.1.0.22
                                 10.1.0.22             -       100     0       65000 65022 i
 * >Ec    RD: 10.1.0.2:1 ethernet-segment 0000:0000:0000:2122:0020 10.1.0.2
                                 10.1.0.2              -       100     0       65000 65002 i
 *  ec    RD: 10.1.0.2:1 ethernet-segment 0000:0000:0000:2122:0020 10.1.0.2
                                 10.1.0.2              -       100     0       65000 65002 i
 * >Ec    RD: 10.1.0.22:1 ethernet-segment 0000:0000:0000:2122:0020 10.1.0.22
                                 10.1.0.22             -       100     0       65000 65022 i
 *  ec    RD: 10.1.0.22:1 ethernet-segment 0000:0000:0000:2122:0020 10.1.0.22
                                 10.1.0.22             -       100     0       65000 65022 i
 * >Ec    RD: 10.1.0.3:1 ip-prefix 10.4.10.0/24
                                 10.1.0.3              -       100     0       65000 65003 i
 *  ec    RD: 10.1.0.3:1 ip-prefix 10.4.10.0/24
                                 10.1.0.3              -       100     0       65000 65003 i
 * >      RD: 10.1.0.11:1 ip-prefix 10.4.10.0/24
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.2:1 ip-prefix 10.4.20.0/24
                                 10.1.0.2              -       100     0       65000 65002 i
 *  ec    RD: 10.1.0.2:1 ip-prefix 10.4.20.0/24
                                 10.1.0.2              -       100     0       65000 65002 i
 * >Ec    RD: 10.1.0.3:1 ip-prefix 10.4.20.0/24
                                 10.1.0.3              -       100     0       65000 65003 i
 *  ec    RD: 10.1.0.3:1 ip-prefix 10.4.20.0/24
                                 10.1.0.3              -       100     0       65000 65003 i
 * >Ec    RD: 10.1.0.22:1 ip-prefix 10.4.20.0/24
                                 10.1.0.22             -       100     0       65000 65022 i
 *  ec    RD: 10.1.0.22:1 ip-prefix 10.4.20.0/24
                                 10.1.0.22             -       100     0       65000 65022 i
Leaf-1-2# show ip route vrf BLUE

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
 B E      10.4.20.0/24 [200/0] via VTEP 10.1.0.3 VNI 10001 router-mac 50:00:00:15:f4:e8 local-interface Vxlan1
                               via VTEP 10.1.0.2 VNI 10001 router-mac 50:00:00:03:37:66 local-interface Vxlan1

Leaf-1-2#show mac address-table
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
  10    0000.0000.0001    STATIC      Po999
  10    5000.00d5.5dc0    STATIC      Po999
  10    aabb.cc80.6000    DYNAMIC     Po10       1       0:01:07 ago
  10    aabb.cc80.8000    DYNAMIC     Vx1        1       0:01:06 ago
4090    0000.0000.0001    STATIC      Cpu
4091    0000.0000.0001    STATIC      Cpu
4094    0000.0000.0001    STATIC      Cpu
4094    5000.0003.3766    DYNAMIC     Vx1        1       1:13:32 ago
4094    5000.0015.f4e8    DYNAMIC     Vx1        1       1:13:32 ago
4094    5000.00af.d3f6    DYNAMIC     Vx1        1       1:13:32 ago
Total Mac Addresses for this criterion: 10

          Multicast Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       ----        -----
Total Mac Addresses for this criterion: 0
Leaf-1-2#show vxlan address-table
          Vxlan Mac Address Table
----------------------------------------------------------------------

VLAN  Mac Address     Type      Prt  VTEP             Moves   Last Move
----  -----------     ----      ---  ----             -----   ---------
  10  aabb.cc80.8000  EVPN      Vx1  10.1.0.3         1       0:01:09 ago
4094  5000.0003.3766  EVPN      Vx1  10.1.0.2         1       1:13:35 ago
4094  5000.0015.f4e8  EVPN      Vx1  10.1.0.3         1       1:13:35 ago
4094  5000.00af.d3f6  EVPN      Vx1  10.1.0.22        1       1:13:35 ago
Total Remote Mac Addresses for this criterion: 4
Leaf-1-2#show mlag
MLAG Configuration:
domain-id                          :            mlag-999
local-interface                    :            Vlan4090
peer-address                       :        10.199.250.1
peer-link                          :     Port-Channel999
peer-config                        :          consistent

MLAG Status:
state                              :              Active
negotiation status                 :           Connected
peer-link status                   :                  Up
local-int status                   :                  Up
system-id                          :   52:00:00:88:fe:27
dual-primary detection             :            Disabled
dual-primary interface errdisabled :               False

MLAG Ports:
Disabled                           :                   0
Configured                         :                   0
Inactive                           :                   0
Active-partial                     :                   0
Active-full                        :                   1

Leaf-1-2#show mlag interfaces 10
                                                                   local/remote
  mlag      desc                 state       local       remote          status
--------- -------------- ---------------- ----------- ------------ ------------
    10      to Host-1      active-full        Po10         Po10           up/up
Leaf-1-2#
Leaf-1-2#sh lacp peer
State: A = Active, P = Passive; S=ShortTimeout, L=LongTimeout;
       G = Aggregable, I = Individual; s+=InSync, s-=OutOfSync;
       C = Collecting, X = state machine expired,
       D = Distributing, d = default neighbor state
                 |                        Partner
 Port    Status  | Sys-id                    Port#   State     OperKey  PortPri
------ ----------|------------------------- ------- --------- --------- -------
Port Channel Port-Channel10*:
 Et3     Bundled | 8000,aa-bb-cc-80-60-00        2   ALGs+CD    0x000a    32768
Port Channel Port-Channel999:
 Et4     Bundled | 8000,50-00-00-d5-5d-c0        4   ALGs+CD    0x03e7    32768
 Et5     Bundled | 8000,50-00-00-d5-5d-c0        5   ALGs+CD    0x03e7    32768

* - Only local interfaces for MLAGs are displayed. Connect to the peer to
    see the state for peer interfaces.
Leaf-1-2#

```
</details>


<details>
<summary> на Leaf-2-1 </summary>
  
```
Leaf-2-1#show bgp summary
BGP summary information for VRF default
Router identifier 10.0.0.2, local AS number 65002
Neighbor          AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc
-------- ----------- ------------- ----------------------- -------------- ---------- ----------
10.0.1.0       65000 Established   IPv4 Unicast            Negotiated             10         10
10.0.1.0       65000 Established   L2VPN EVPN              Negotiated             18         18
10.0.2.0       65000 Established   IPv4 Unicast            Negotiated             10         10
10.0.2.0       65000 Established   L2VPN EVPN              Negotiated             18         18
10.2.1.2       65000 Established   IPv4 Unicast            Negotiated             10         10
10.2.2.2       65000 Established   IPv4 Unicast            Negotiated             10         10
Leaf-2-1#show ip bgp
BGP routing table information for VRF default
Router identifier 10.0.0.2, local AS number 65002
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >Ec    10.0.0.1/32            10.2.1.2              0       -          100     0       65000 65001 i
 *  ec    10.0.0.1/32            10.2.2.2              0       -          100     0       65000 65001 i
          10.0.0.1/32            10.2.1.1              0       -          100     0       65000 65001 i
          10.0.0.1/32            10.2.2.1              0       -          100     0       65000 65001 i
 * >      10.0.0.2/32            -                     -       -          -       0       i
 * >Ec    10.0.0.3/32            10.2.2.2              0       -          100     0       65000 65003 i
 *  ec    10.0.0.3/32            10.2.1.2              0       -          100     0       65000 65003 i
          10.0.0.3/32            10.2.2.5              0       -          100     0       65000 65003 i
          10.0.0.3/32            10.2.1.5              0       -          100     0       65000 65003 i
 * >Ec    10.0.0.11/32           10.2.1.2              0       -          100     0       65000 65001 i
 *  ec    10.0.0.11/32           10.2.2.2              0       -          100     0       65000 65001 i
          10.0.0.11/32           10.2.1.1              0       -          100     0       65000 65001 i
          10.0.0.11/32           10.2.2.1              0       -          100     0       65000 65001 i
 * >Ec    10.0.0.22/32           10.2.1.2              0       -          100     0       65000 65022 i
 *  ec    10.0.0.22/32           10.2.2.2              0       -          100     0       65000 65022 i
          10.0.0.22/32           10.2.1.23             0       -          100     0       65000 65022 i
          10.0.0.22/32           10.2.2.23             0       -          100     0       65000 65022 i
 * >      10.0.1.0/32            10.2.1.2              0       -          100     0       65000 i
 *        10.0.1.0/32            10.0.1.0              0       -          100     0       65000 i
 * >      10.0.2.0/32            10.2.2.2              0       -          100     0       65000 i
 *        10.0.2.0/32            10.0.2.0              0       -          100     0       65000 i
 * >Ec    10.1.0.1/32            10.2.1.2              0       -          100     0       65000 65001 i
 *  ec    10.1.0.1/32            10.2.2.2              0       -          100     0       65000 65001 i
          10.1.0.1/32            10.2.1.1              0       -          100     0       65000 65001 i
          10.1.0.1/32            10.2.2.1              0       -          100     0       65000 65001 i
 * >      10.1.0.2/32            -                     -       -          -       0       i
 * >Ec    10.1.0.3/32            10.2.2.2              0       -          100     0       65000 65003 i
 *  ec    10.1.0.3/32            10.2.1.2              0       -          100     0       65000 65003 i
          10.1.0.3/32            10.2.2.5              0       -          100     0       65000 65003 i
          10.1.0.3/32            10.2.1.5              0       -          100     0       65000 65003 i
 * >Ec    10.1.0.11/32           10.2.1.2              0       -          100     0       65000 65001 i
 *  ec    10.1.0.11/32           10.2.2.2              0       -          100     0       65000 65001 i
          10.1.0.11/32           10.2.1.1              0       -          100     0       65000 65001 i
          10.1.0.11/32           10.2.2.1              0       -          100     0       65000 65001 i
 * >Ec    10.1.0.22/32           10.2.1.2              0       -          100     0       65000 65022 i
 *  ec    10.1.0.22/32           10.2.2.2              0       -          100     0       65000 65022 i
          10.1.0.22/32           10.2.1.23             0       -          100     0       65000 65022 i
          10.1.0.22/32           10.2.2.23             0       -          100     0       65000 65022 i
 * >      10.1.1.0/32            10.2.1.2              0       -          100     0       65000 i
 *        10.1.1.0/32            10.0.1.0              0       -          100     0       65000 i
 * >      10.1.2.0/32            10.2.2.2              0       -          100     0       65000 i
 *        10.1.2.0/32            10.0.2.0              0       -          100     0       65000 i
 Leaf-2-1#
Leaf-2-1#
Leaf-2-1#show bgp evpn
BGP routing table information for VRF default
Router identifier 10.0.0.2, local AS number 65002
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 65002:10020 auto-discovery 0 0000:0000:0000:2122:0020
                                 -                     -       -       0       i
 * >      RD: 10.1.0.2:1 auto-discovery 0000:0000:0000:2122:0020
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.22:1 auto-discovery 0000:0000:0000:2122:0020
                                 10.1.0.22             -       100     0       65000 65022 i
 *  ec    RD: 10.1.0.22:1 auto-discovery 0000:0000:0000:2122:0020
                                 10.1.0.22             -       100     0       65000 65022 i
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
 * >Ec    RD: 65001:10010 imet 10.1.0.11
                                 10.1.0.11             -       100     0       65000 65001 i
 *  ec    RD: 65001:10010 imet 10.1.0.11
                                 10.1.0.11             -       100     0       65000 65001 i
 * >Ec    RD: 65002:10020 imet 10.1.0.22
                                 10.1.0.22             -       100     0       65000 65022 i
 *  ec    RD: 65002:10020 imet 10.1.0.22
                                 10.1.0.22             -       100     0       65000 65022 i
 * >      RD: 10.1.0.2:1 ethernet-segment 0000:0000:0000:2122:0020 10.1.0.2
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.22:1 ethernet-segment 0000:0000:0000:2122:0020 10.1.0.22
                                 10.1.0.22             -       100     0       65000 65022 i
 *  ec    RD: 10.1.0.22:1 ethernet-segment 0000:0000:0000:2122:0020 10.1.0.22
                                 10.1.0.22             -       100     0       65000 65022 i
 * >Ec    RD: 10.1.0.1:1 ip-prefix 10.4.10.0/24
                                 10.1.0.1              -       100     0       65000 65001 i
 *  ec    RD: 10.1.0.1:1 ip-prefix 10.4.10.0/24
                                 10.1.0.1              -       100     0       65000 65001 i
 * >Ec    RD: 10.1.0.3:1 ip-prefix 10.4.10.0/24
                                 10.1.0.3              -       100     0       65000 65003 i
 *  ec    RD: 10.1.0.3:1 ip-prefix 10.4.10.0/24
                                 10.1.0.3              -       100     0       65000 65003 i
 * >Ec    RD: 10.1.0.11:1 ip-prefix 10.4.10.0/24
                                 10.1.0.11             -       100     0       65000 65001 i
 *  ec    RD: 10.1.0.11:1 ip-prefix 10.4.10.0/24
                                 10.1.0.11             -       100     0       65000 65001 i
 * >      RD: 10.1.0.2:1 ip-prefix 10.4.20.0/24
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.3:1 ip-prefix 10.4.20.0/24
                                 10.1.0.3              -       100     0       65000 65003 i
 *  ec    RD: 10.1.0.3:1 ip-prefix 10.4.20.0/24
                                 10.1.0.3              -       100     0       65000 65003 i
 * >Ec    RD: 10.1.0.22:1 ip-prefix 10.4.20.0/24
                                 10.1.0.22             -       100     0       65000 65022 i
 *  ec    RD: 10.1.0.22:1 ip-prefix 10.4.20.0/24
                                 10.1.0.22             -       100     0       65000 65022 i
Leaf-2-1#
Leaf-2-1#
Leaf-2-1#show ip route vrf BLUE

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
 B E      10.4.10.0/24 [200/0] via VTEP 10.1.0.3 VNI 10001 router-mac 50:00:00:15:f4:e8 local-interface Vxlan1
                               via VTEP 10.1.0.11 VNI 10001 router-mac 50:00:00:88:fe:27 local-interface Vxlan1
 B E      10.4.20.40/32 [200/0] via VTEP 10.1.0.3 VNI 10001 router-mac 50:00:00:15:f4:e8 local-interface Vxlan1
 C        10.4.20.0/24 is directly connected, Vlan20

Leaf-2-1#   show mac address-table
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
  20    0000.0000.0001    STATIC      Cpu
  20    aabb.cc80.7000    DYNAMIC     Po20       1       0:05:43 ago
  20    aabb.cc80.9000    DYNAMIC     Vx1        1       0:02:25 ago
4094    0000.0000.0001    STATIC      Cpu
4094    5000.0015.f4e8    DYNAMIC     Vx1        1       2:35:44 ago
4094    5000.0088.fe27    DYNAMIC     Vx1        1       1:15:03 ago
4094    5000.00af.d3f6    DYNAMIC     Vx1        1       2:24:50 ago
4094    5000.00d5.5dc0    DYNAMIC     Vx1        1       1:25:41 ago
Total Mac Addresses for this criterion: 8

          Multicast Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       ----        -----
Total Mac Addresses for this criterion: 0
Leaf-2-1# show vxlan address-table
          Vxlan Mac Address Table
----------------------------------------------------------------------

VLAN  Mac Address     Type      Prt  VTEP             Moves   Last Move
----  -----------     ----      ---  ----             -----   ---------
  20  aabb.cc80.9000  EVPN      Vx1  10.1.0.3         1       0:02:30 ago
4094  5000.0015.f4e8  EVPN      Vx1  10.1.0.3         1       2:35:48 ago
4094  5000.0088.fe27  EVPN      Vx1  10.1.0.11        1       1:15:07 ago
4094  5000.00af.d3f6  EVPN      Vx1  10.1.0.22        1       2:24:54 ago
4094  5000.00d5.5dc0  EVPN      Vx1  10.1.0.1         1       1:25:46 ago
Total Remote Mac Addresses for this criterion: 5
Leaf-2-1#
Leaf-2-1# show lacp peer
State: A = Active, P = Passive; S=ShortTimeout, L=LongTimeout;
       G = Aggregable, I = Individual; s+=InSync, s-=OutOfSync;
       C = Collecting, X = state machine expired,
       D = Distributing, d = default neighbor state
                 |                        Partner
 Port    Status  | Sys-id                    Port#   State     OperKey  PortPri
------ ----------|------------------------- ------- --------- --------- -------
Port Channel Port-Channel20:
 Et3     Bundled | 8000,aa-bb-cc-80-70-00        1   ALGs+CD    0x0014    32768

Leaf-2-1#
Leaf-2-1#sh bgp evpn instance
EVPN instance: VLAN 20
  Route distinguisher: 65002:10020
  Route target import: Route-Target-AS:20:10020
  Route target export: Route-Target-AS:20:10020
  Service interface: VLAN-based
  Local VXLAN IP address: 10.1.0.2
  VXLAN: enabled
  MPLS: disabled
  Local ethernet segment:
    ESI: 0000:0000:0000:2122:0020
      Interface: Port-Channel20
      Mode: all-active
      State: up
      ES-Import RT: 00:00:21:22:00:20
      DF election algorithm: preference
      Designated forwarder: 10.1.0.2
      Non-Designated forwarder: 10.1.0.22
Leaf-2-1#

```
</details>


<details>
<summary> на Leaf-2-2 </summary>
  
```
Leaf-2-2#show bgp summary
BGP summary information for VRF default
Router identifier 10.0.0.22, local AS number 65022
Neighbor           AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc
--------- ----------- ------------- ----------------------- -------------- ---------- ----------
10.0.1.0        65000 Established   IPv4 Unicast            Negotiated             10         10
10.0.1.0        65000 Established   L2VPN EVPN              Negotiated             21         21
10.0.2.0        65000 Established   IPv4 Unicast            Negotiated             10         10
10.0.2.0        65000 Established   L2VPN EVPN              Negotiated             21         21
10.2.1.22       65000 Established   IPv4 Unicast            Negotiated             10         10
10.2.2.22       65000 Established   IPv4 Unicast            Negotiated             10         10
Leaf-2-2#
Leaf-2-2#
Leaf-2-2#show ip bgp
BGP routing table information for VRF default
Router identifier 10.0.0.22, local AS number 65022
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >Ec    10.0.0.1/32            10.2.1.22             0       -          100     0       65000 65001 i
 *  ec    10.0.0.1/32            10.2.2.22             0       -          100     0       65000 65001 i
          10.0.0.1/32            10.2.1.1              0       -          100     0       65000 65001 i
          10.0.0.1/32            10.2.2.1              0       -          100     0       65000 65001 i
 * >Ec    10.0.0.2/32            10.2.1.22             0       -          100     0       65000 65002 i
 *  ec    10.0.0.2/32            10.2.2.22             0       -          100     0       65000 65002 i
          10.0.0.2/32            10.2.1.3              0       -          100     0       65000 65002 i
          10.0.0.2/32            10.2.2.3              0       -          100     0       65000 65002 i
 * >Ec    10.0.0.3/32            10.2.1.22             0       -          100     0       65000 65003 i
 *  ec    10.0.0.3/32            10.2.2.22             0       -          100     0       65000 65003 i
          10.0.0.3/32            10.2.1.5              0       -          100     0       65000 65003 i
          10.0.0.3/32            10.2.2.5              0       -          100     0       65000 65003 i
 * >Ec    10.0.0.11/32           10.2.2.22             0       -          100     0       65000 65001 i
 *  ec    10.0.0.11/32           10.2.1.22             0       -          100     0       65000 65001 i
          10.0.0.11/32           10.2.1.1              0       -          100     0       65000 65001 i
          10.0.0.11/32           10.2.2.1              0       -          100     0       65000 65001 i
 * >      10.0.0.22/32           -                     -       -          -       0       i
 * >      10.0.1.0/32            10.2.1.22             0       -          100     0       65000 i
 *        10.0.1.0/32            10.0.1.0              0       -          100     0       65000 i
 * >      10.0.2.0/32            10.2.2.22             0       -          100     0       65000 i
 *        10.0.2.0/32            10.0.2.0              0       -          100     0       65000 i
 * >Ec    10.1.0.1/32            10.2.1.22             0       -          100     0       65000 65001 i
 *  ec    10.1.0.1/32            10.2.2.22             0       -          100     0       65000 65001 i
          10.1.0.1/32            10.2.1.1              0       -          100     0       65000 65001 i
          10.1.0.1/32            10.2.2.1              0       -          100     0       65000 65001 i
 * >Ec    10.1.0.2/32            10.2.1.22             0       -          100     0       65000 65002 i
 *  ec    10.1.0.2/32            10.2.2.22             0       -          100     0       65000 65002 i
          10.1.0.2/32            10.2.1.3              0       -          100     0       65000 65002 i
          10.1.0.2/32            10.2.2.3              0       -          100     0       65000 65002 i
 * >Ec    10.1.0.3/32            10.2.1.22             0       -          100     0       65000 65003 i
 *  ec    10.1.0.3/32            10.2.2.22             0       -          100     0       65000 65003 i
          10.1.0.3/32            10.2.1.5              0       -          100     0       65000 65003 i
          10.1.0.3/32            10.2.2.5              0       -          100     0       65000 65003 i
 * >Ec    10.1.0.11/32           10.2.2.22             0       -          100     0       65000 65001 i
 *  ec    10.1.0.11/32           10.2.1.22             0       -          100     0       65000 65001 i
          10.1.0.11/32           10.2.1.1              0       -          100     0       65000 65001 i
          10.1.0.11/32           10.2.2.1              0       -          100     0       65000 65001 i
 * >      10.1.0.22/32           -                     -       -          -       0       i
 * >      10.1.1.0/32            10.2.1.22             0       -          100     0       65000 i
 *        10.1.1.0/32            10.0.1.0              0       -          100     0       65000 i
 * >      10.1.2.0/32            10.2.2.22             0       -          100     0       65000 i
 *        10.1.2.0/32            10.0.2.0              0       -          100     0       65000 i
Leaf-2-2#  show bgp evpn
BGP routing table information for VRF default
Router identifier 10.0.0.22, local AS number 65022
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 65002:10020 auto-discovery 0 0000:0000:0000:2122:0020
                                 -                     -       -       0       i
 *  Ec    RD: 65002:10020 auto-discovery 0 0000:0000:0000:2122:0020
                                 10.1.0.2              -       100     0       65000 65002 i
 *  ec    RD: 65002:10020 auto-discovery 0 0000:0000:0000:2122:0020
                                 10.1.0.2              -       100     0       65000 65002 i
 * >Ec    RD: 10.1.0.2:1 auto-discovery 0000:0000:0000:2122:0020
                                 10.1.0.2              -       100     0       65000 65002 i
 *  ec    RD: 10.1.0.2:1 auto-discovery 0000:0000:0000:2122:0020
                                 10.1.0.2              -       100     0       65000 65002 i
 * >      RD: 10.1.0.22:1 auto-discovery 0000:0000:0000:2122:0020
                                 -                     -       -       0       i
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
 *        RD: 65002:10020 mac-ip aabb.cc80.7000 10.4.20.20
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
 * >Ec    RD: 65001:10010 imet 10.1.0.11
                                 10.1.0.11             -       100     0       65000 65001 i
 *  ec    RD: 65001:10010 imet 10.1.0.11
                                 10.1.0.11             -       100     0       65000 65001 i
 * >      RD: 65002:10020 imet 10.1.0.22
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.2:1 ethernet-segment 0000:0000:0000:2122:0020 10.1.0.2
                                 10.1.0.2              -       100     0       65000 65002 i
 *  ec    RD: 10.1.0.2:1 ethernet-segment 0000:0000:0000:2122:0020 10.1.0.2
                                 10.1.0.2              -       100     0       65000 65002 i
 * >      RD: 10.1.0.22:1 ethernet-segment 0000:0000:0000:2122:0020 10.1.0.22
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.1:1 ip-prefix 10.4.10.0/24
                                 10.1.0.1              -       100     0       65000 65001 i
 *  ec    RD: 10.1.0.1:1 ip-prefix 10.4.10.0/24
                                 10.1.0.1              -       100     0       65000 65001 i
 * >Ec    RD: 10.1.0.3:1 ip-prefix 10.4.10.0/24
                                 10.1.0.3              -       100     0       65000 65003 i
 *  ec    RD: 10.1.0.3:1 ip-prefix 10.4.10.0/24
                                 10.1.0.3              -       100     0       65000 65003 i
 * >Ec    RD: 10.1.0.11:1 ip-prefix 10.4.10.0/24
                                 10.1.0.11             -       100     0       65000 65001 i
 *  ec    RD: 10.1.0.11:1 ip-prefix 10.4.10.0/24
                                 10.1.0.11             -       100     0       65000 65001 i
 * >Ec    RD: 10.1.0.2:1 ip-prefix 10.4.20.0/24
                                 10.1.0.2              -       100     0       65000 65002 i
 *  ec    RD: 10.1.0.2:1 ip-prefix 10.4.20.0/24
                                 10.1.0.2              -       100     0       65000 65002 i
 * >Ec    RD: 10.1.0.3:1 ip-prefix 10.4.20.0/24
                                 10.1.0.3              -       100     0       65000 65003 i
 *  ec    RD: 10.1.0.3:1 ip-prefix 10.4.20.0/24
                                 10.1.0.3              -       100     0       65000 65003 i
 * >      RD: 10.1.0.22:1 ip-prefix 10.4.20.0/24
                                 -                     -       -       0       i
Leaf-2-2#  show ip route vrf BLUE

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
 B E      10.4.10.0/24 [200/0] via VTEP 10.1.0.3 VNI 10001 router-mac 50:00:00:15:f4:e8 local-interface Vxlan1
                               via VTEP 10.1.0.11 VNI 10001 router-mac 50:00:00:88:fe:27 local-interface Vxlan1
 B E      10.4.20.40/32 [200/0] via VTEP 10.1.0.3 VNI 10001 router-mac 50:00:00:15:f4:e8 local-interface Vxlan1
 C        10.4.20.0/24 is directly connected, Vlan20

Leaf-2-2#  show mac address-table
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
   1    0000.0000.0001    STATIC      Cpu
  20    0000.0000.0001    STATIC      Cpu
  20    aabb.cc80.7000    DYNAMIC     Po20       1       0:09:29 ago
  20    aabb.cc80.9000    DYNAMIC     Vx1        1       0:06:12 ago
4094    0000.0000.0001    STATIC      Cpu
4094    5000.0003.3766    DYNAMIC     Vx1        1       2:28:36 ago
4094    5000.0015.f4e8    DYNAMIC     Vx1        1       2:28:35 ago
4094    5000.0088.fe27    DYNAMIC     Vx1        1       1:18:49 ago
4094    5000.00d5.5dc0    DYNAMIC     Vx1        1       1:29:28 ago
Total Mac Addresses for this criterion: 9

          Multicast Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       ----        -----
Total Mac Addresses for this criterion: 0
Leaf-2-2#  show vxlan address-table
          Vxlan Mac Address Table
----------------------------------------------------------------------

VLAN  Mac Address     Type      Prt  VTEP             Moves   Last Move
----  -----------     ----      ---  ----             -----   ---------
  20  aabb.cc80.7000  EVPN      Vx1  0.0.0.0          1       0:09:33 ago
  20  aabb.cc80.9000  EVPN      Vx1  10.1.0.3         1       0:06:15 ago
4094  5000.0003.3766  EVPN      Vx1  10.1.0.2         1       2:28:39 ago
4094  5000.0015.f4e8  EVPN      Vx1  10.1.0.3         1       2:28:39 ago
4094  5000.0088.fe27  EVPN      Vx1  10.1.0.11        1       1:18:52 ago
4094  5000.00d5.5dc0  EVPN      Vx1  10.1.0.1         1       1:29:31 ago
Total Remote Mac Addresses for this criterion: 6
Leaf-2-2#
Leaf-2-2#show lacp peer
State: A = Active, P = Passive; S=ShortTimeout, L=LongTimeout;
       G = Aggregable, I = Individual; s+=InSync, s-=OutOfSync;
       C = Collecting, X = state machine expired,
       D = Distributing, d = default neighbor state
                 |                        Partner
 Port    Status  | Sys-id                    Port#   State     OperKey  PortPri
------ ----------|------------------------- ------- --------- --------- -------
Port Channel Port-Channel20:
 Et3     Bundled | 8000,aa-bb-cc-80-70-00        2   ALGs+CD    0x0014    32768

Leaf-2-2#
Leaf-2-2#sh bgp evpn instance
EVPN instance: VLAN 20
  Route distinguisher: 65002:10020
  Route target import: Route-Target-AS:20:10020
  Route target export: Route-Target-AS:20:10020
  Service interface: VLAN-based
  Local VXLAN IP address: 10.1.0.22
  VXLAN: enabled
  MPLS: disabled
  Local ethernet segment:
    ESI: 0000:0000:0000:2122:0020
      Interface: Port-Channel20
      Mode: all-active
      State: up
      ES-Import RT: 00:00:21:22:00:20
      DF election algorithm: preference
      Designated forwarder: 10.1.0.2
      Non-Designated forwarder: 10.1.0.22

```
</details>

<details>
<summary> на Leaf-3 </summary>
  
```
Leaf-3#
Leaf-3#sh bgp summary
BGP summary information for VRF default
Router identifier 10.0.0.3, local AS number 65003
Neighbor          AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc
-------- ----------- ------------- ----------------------- -------------- ---------- ----------
10.0.1.0       65000 Established   IPv4 Unicast            Negotiated             10         10
10.0.1.0       65000 Established   L2VPN EVPN              Negotiated             17         17
10.0.2.0       65000 Established   IPv4 Unicast            Negotiated             10         10
10.0.2.0       65000 Established   L2VPN EVPN              Negotiated             17         17
10.2.1.4       65000 Established   IPv4 Unicast            Negotiated             10         10
10.2.2.4       65000 Established   IPv4 Unicast            Negotiated             10         10
Leaf-3#
Leaf-3#show ip bgp
BGP routing table information for VRF default
Router identifier 10.0.0.3, local AS number 65003
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >Ec    10.0.0.1/32            10.2.1.4              0       -          100     0       65000 65001 i
 *  ec    10.0.0.1/32            10.2.2.4              0       -          100     0       65000 65001 i
          10.0.0.1/32            10.2.1.1              0       -          100     0       65000 65001 i
          10.0.0.1/32            10.2.2.1              0       -          100     0       65000 65001 i
 * >Ec    10.0.0.2/32            10.2.2.4              0       -          100     0       65000 65002 i
 *  ec    10.0.0.2/32            10.2.1.4              0       -          100     0       65000 65002 i
          10.0.0.2/32            10.2.1.3              0       -          100     0       65000 65002 i
          10.0.0.2/32            10.2.2.3              0       -          100     0       65000 65002 i
 * >      10.0.0.3/32            -                     -       -          -       0       i
 * >Ec    10.0.0.11/32           10.2.1.4              0       -          100     0       65000 65001 i
 *  ec    10.0.0.11/32           10.2.2.4              0       -          100     0       65000 65001 i
          10.0.0.11/32           10.2.1.1              0       -          100     0       65000 65001 i
          10.0.0.11/32           10.2.2.1              0       -          100     0       65000 65001 i
 * >Ec    10.0.0.22/32           10.2.1.4              0       -          100     0       65000 65022 i
 *  ec    10.0.0.22/32           10.2.2.4              0       -          100     0       65000 65022 i
          10.0.0.22/32           10.2.1.23             0       -          100     0       65000 65022 i
          10.0.0.22/32           10.2.2.23             0       -          100     0       65000 65022 i
 * >      10.0.1.0/32            10.2.1.4              0       -          100     0       65000 i
 *        10.0.1.0/32            10.0.1.0              0       -          100     0       65000 i
 * >      10.0.2.0/32            10.2.2.4              0       -          100     0       65000 i
 *        10.0.2.0/32            10.0.2.0              0       -          100     0       65000 i
 * >Ec    10.1.0.1/32            10.2.1.4              0       -          100     0       65000 65001 i
 *  ec    10.1.0.1/32            10.2.2.4              0       -          100     0       65000 65001 i
          10.1.0.1/32            10.2.1.1              0       -          100     0       65000 65001 i
          10.1.0.1/32            10.2.2.1              0       -          100     0       65000 65001 i
 * >Ec    10.1.0.2/32            10.2.2.4              0       -          100     0       65000 65002 i
 *  ec    10.1.0.2/32            10.2.1.4              0       -          100     0       65000 65002 i
          10.1.0.2/32            10.2.1.3              0       -          100     0       65000 65002 i
          10.1.0.2/32            10.2.2.3              0       -          100     0       65000 65002 i
 * >      10.1.0.3/32            -                     -       -          -       0       i
 * >Ec    10.1.0.11/32           10.2.1.4              0       -          100     0       65000 65001 i
 *  ec    10.1.0.11/32           10.2.2.4              0       -          100     0       65000 65001 i
          10.1.0.11/32           10.2.1.1              0       -          100     0       65000 65001 i
          10.1.0.11/32           10.2.2.1              0       -          100     0       65000 65001 i
 * >Ec    10.1.0.22/32           10.2.1.4              0       -          100     0       65000 65022 i
 *  ec    10.1.0.22/32           10.2.2.4              0       -          100     0       65000 65022 i
          10.1.0.22/32           10.2.1.23             0       -          100     0       65000 65022 i
          10.1.0.22/32           10.2.2.23             0       -          100     0       65000 65022 i
 * >      10.1.1.0/32            10.2.1.4              0       -          100     0       65000 i
 *        10.1.1.0/32            10.0.1.0              0       -          100     0       65000 i
 * >      10.1.2.0/32            10.2.2.4              0       -          100     0       65000 i
 *        10.1.2.0/32            10.0.2.0              0       -          100     0       65000 i
Leaf-3#
Leaf-3#  show bgp evpn
BGP routing table information for VRF default
Router identifier 10.0.0.3, local AS number 65003
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 65002:10020 auto-discovery 0 0000:0000:0000:2122:0020
                                 10.1.0.2              -       100     0       65000 65002 i
 *  ec    RD: 65002:10020 auto-discovery 0 0000:0000:0000:2122:0020
                                 10.1.0.2              -       100     0       65000 65002 i
 * >Ec    RD: 10.1.0.2:1 auto-discovery 0000:0000:0000:2122:0020
                                 10.1.0.2              -       100     0       65000 65002 i
 *  ec    RD: 10.1.0.2:1 auto-discovery 0000:0000:0000:2122:0020
                                 10.1.0.2              -       100     0       65000 65002 i
 * >Ec    RD: 10.1.0.22:1 auto-discovery 0000:0000:0000:2122:0020
                                 10.1.0.22             -       100     0       65000 65022 i
 *  ec    RD: 10.1.0.22:1 auto-discovery 0000:0000:0000:2122:0020
                                 10.1.0.22             -       100     0       65000 65022 i
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
 * >Ec    RD: 65001:10010 imet 10.1.0.11
                                 10.1.0.11             -       100     0       65000 65001 i
 *  ec    RD: 65001:10010 imet 10.1.0.11
                                 10.1.0.11             -       100     0       65000 65001 i
 * >Ec    RD: 65002:10020 imet 10.1.0.22
                                 10.1.0.22             -       100     0       65000 65022 i
 *  ec    RD: 65002:10020 imet 10.1.0.22
                                 10.1.0.22             -       100     0       65000 65022 i
 * >Ec    RD: 10.1.0.2:1 ethernet-segment 0000:0000:0000:2122:0020 10.1.0.2
                                 10.1.0.2              -       100     0       65000 65002 i
 *  ec    RD: 10.1.0.2:1 ethernet-segment 0000:0000:0000:2122:0020 10.1.0.2
                                 10.1.0.2              -       100     0       65000 65002 i
 * >Ec    RD: 10.1.0.22:1 ethernet-segment 0000:0000:0000:2122:0020 10.1.0.22
                                 10.1.0.22             -       100     0       65000 65022 i
 *  ec    RD: 10.1.0.22:1 ethernet-segment 0000:0000:0000:2122:0020 10.1.0.22
                                 10.1.0.22             -       100     0       65000 65022 i
 * >Ec    RD: 10.1.0.1:1 ip-prefix 10.4.10.0/24
                                 10.1.0.1              -       100     0       65000 65001 i
 *  ec    RD: 10.1.0.1:1 ip-prefix 10.4.10.0/24
                                 10.1.0.1              -       100     0       65000 65001 i
 * >      RD: 10.1.0.3:1 ip-prefix 10.4.10.0/24
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.11:1 ip-prefix 10.4.10.0/24
                                 10.1.0.11             -       100     0       65000 65001 i
 *  ec    RD: 10.1.0.11:1 ip-prefix 10.4.10.0/24
                                 10.1.0.11             -       100     0       65000 65001 i
 * >Ec    RD: 10.1.0.2:1 ip-prefix 10.4.20.0/24
                                 10.1.0.2              -       100     0       65000 65002 i
 *  ec    RD: 10.1.0.2:1 ip-prefix 10.4.20.0/24
                                 10.1.0.2              -       100     0       65000 65002 i
 * >      RD: 10.1.0.3:1 ip-prefix 10.4.20.0/24
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.22:1 ip-prefix 10.4.20.0/24
                                 10.1.0.22             -       100     0       65000 65022 i
 *  ec    RD: 10.1.0.22:1 ip-prefix 10.4.20.0/24
                                 10.1.0.22             -       100     0       65000 65022 i
Leaf-3#
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

Leaf-3#    show mac address-table
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
   1    0000.0000.0001    STATIC      Cpu
  10    0000.0000.0001    STATIC      Cpu
  10    aabb.cc80.6000    DYNAMIC     Vx1        1       0:08:02 ago
  10    aabb.cc80.8000    DYNAMIC     Et3        1       0:08:02 ago
  20    0000.0000.0001    STATIC      Cpu
  20    aabb.cc80.7000    DYNAMIC     Vx1        1       0:11:12 ago
  20    aabb.cc80.9000    DYNAMIC     Et4        1       0:07:55 ago
4094    0000.0000.0001    STATIC      Cpu
4094    5000.0003.3766    DYNAMIC     Vx1        1       2:41:12 ago
4094    5000.0088.fe27    DYNAMIC     Vx1        1       1:20:32 ago
4094    5000.00af.d3f6    DYNAMIC     Vx1        1       2:30:19 ago
4094    5000.00d5.5dc0    DYNAMIC     Vx1        1       1:31:10 ago
Total Mac Addresses for this criterion: 12

          Multicast Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       ----        -----
Total Mac Addresses for this criterion: 0
Leaf-3#
Leaf-3#show vxlan address-table
          Vxlan Mac Address Table
----------------------------------------------------------------------

VLAN  Mac Address     Type      Prt  VTEP             Moves   Last Move
----  -----------     ----      ---  ----             -----   ---------
  10  aabb.cc80.6000  EVPN      Vx1  10.1.0.1         1       0:14:18 ago
  20  aabb.cc80.7000  EVPN      Vx1  10.1.0.2         1       0:16:05 ago
4094  5000.0003.3766  EVPN      Vx1  10.1.0.2         1       3:01:35 ago
4094  5000.0088.fe27  EVPN      Vx1  10.1.0.11        1       1:40:55 ago
4094  5000.00af.d3f6  EVPN      Vx1  10.1.0.22        1       2:50:42 ago
4094  5000.00d5.5dc0  EVPN      Vx1  10.1.0.1         1       1:51:34 ago
Total Remote Mac Addresses for this criterion: 6
Leaf-3#

```
</details>




