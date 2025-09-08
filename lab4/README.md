## ДЗ №4
Underlay. BGP
***
### Цель:
Настроить BGP для Underlay сети.
#### Задачи:
1. Настроить BGP в Underlay сети, для IP связанности между всеми сетевыми устройствами;
2. Зафиксировать в документации - план работы, адресное пространство, схему сети, конфигурацию устройств;
3. Убедиться в наличии IP связанности между устройствами в BGP домене.
***
### Результаты выполнения:
---
В качестве примера будет использован протокол eBGP
- Spine-1 и Spine-2 будут находиться в AS65000;
- Leaf-1, Leaf-2 и Leaf-3 будут находиться в AS65001, AS65002 и AS65003 соответственно.

Ключевые рекомендации:
- Использовать схему ASN, исключающую проблему path-hunting;
- Использовать multipath для ECMP;
- Использовать одну BGP сессиию для передачи информации для множества address families;
- Использовать BFD, оптимизировать таймеры;
- Сохранять конфигурацию минимально необходимой и простой.


#### 1. Схема сети:
   
![](https://github.com/egorvshch/DC-networks-design/blob/main/lab4/net_schema_ebgp_underlay.jpg)
---

#### 2. IP-адресация оборудования для Underlay сети ЦОД:

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
| Server-1	| eth0	| 10.4.1.2	| /24 |
| Server-2	| eth0	| 10.4.2.2	| /24 |
| Server-3	| eth0	| 10.4.3.2	| /24 |
| Server-4	| eth0	| 10.4.4.2	| /24 |

</details>

---

#### 3. Конфигурация оборудования (Arista):

- Общий шаблон конфигурации BGP на Spine-коммутаторах имеет вид:

```
peer-filter LEAFs
   10 match as-range [LEAF_ASN_RANGE] result accept
!
router bgp [LOCAL_ASN]
   router-id [ROUTER_ID]
   timers bgp 3 9
   bgp log-neighbor-changes
   bgp listen range [NEIGHBORS_PREFIX] peer-group LEAF peer-filter LEAFs
   neighbor LEAF peer group
   neighbor LEAF bfd
   neighbor LEAF password 0 otus@1234
   neighbor LEAF send-community
!
end

```

- Общий шаблон конфигурации BGP на Leaf-коммутаторах имеет вид:

```
ip prefix-list DIRECT_CONNECTED
   seq 10 permit [IP_LOOPBACK0]
   seq 20 permit [IP_LOOPBACK1]
!
route-map RM_REDISTRIBUTE_CONNECTED permit 10
   match ip address prefix-list DIRECT_CONNECTED
!
router bgp [LOCAL_ASN]
   router-id [ROUTER_ID]
   timers bgp 3 9
   bgp log-neighbor-changes
   bgp bestpath as-path multipath-relax 
   maximum-paths 2 ecmp 2
   neighbor SPINE peer group
   neighbor SPINE remote-as [SPINE_ASN]
   neighbor SPINE bfd
   neighbor SPINE password 0 otus@1234
   neighbor SPINE send-community
   neighbor [NEIGHBORS1_IP_ADDRESS] peer group SPINE
   neighbor [NEIGHBORS2_IP_ADDRESS] peer group SPINE
   redistribute connected route-map RM_REDISTRIBUTE_CONNECTED
!
end

```

#####Полная конфигурация коммутаторов:

<details>
<summary> Spine-1 </summary>
  
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
service routing protocols model ribd
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
peer-filter LEAFs
   10 match as-range 65001-65003 result accept
!
router bgp 65000
   router-id 10.0.1.0
   timers bgp 3 9
   bgp listen range 10.2.1.0/29 peer-group LEAF peer-filter LEAFs
   neighbor LEAF peer group
   neighbor LEAF bfd
   neighbor LEAF password 7 X924dXED0EODP1A6/K/R/w==
   neighbor LEAF send-community
!
end
Spine-1#

```
</details>

<details>
<summary> Spine-2 </summary>
  
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
service routing protocols model ribd
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
peer-filter LEAFs
   10 match as-range 65001-65003 result accept
!
router bgp 65000
   router-id 10.0.2.0
   timers bgp 3 9
   bgp listen range 10.2.2.0/29 peer-group LEAF peer-filter LEAFs
   neighbor LEAF peer group
   neighbor LEAF bfd
   neighbor LEAF password 7 X924dXED0EODP1A6/K/R/w==
   neighbor LEAF send-community
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
service routing protocols model ribd
!
hostname Leaf-1
!
spanning-tree mode mstp
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
   description to Server-1
   no switchport
   ip address 10.4.1.1/24
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
ip routing
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
   neighbor SPINE peer group
   neighbor SPINE remote-as 65000
   neighbor SPINE bfd
   neighbor SPINE password 7 sWIT5buW1FYPyTvv4vVCmQ==
   neighbor SPINE send-community
   neighbor 10.2.1.0 peer group SPINE
   neighbor 10.2.2.0 peer group SPINE
   redistribute connected route-map RM_REDISTRIBUTE_CONNECTED
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
service routing protocols model ribd
!
hostname Leaf-2
!
spanning-tree mode mstp
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
   description to Server-2
   no switchport
   ip address 10.4.2.1/24
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
ip routing
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
   neighbor SPINE peer group
   neighbor SPINE remote-as 65000
   neighbor SPINE bfd
   neighbor SPINE password 7 sWIT5buW1FYPyTvv4vVCmQ==
   neighbor SPINE send-community
   neighbor 10.2.1.2 peer group SPINE
   neighbor 10.2.2.2 peer group SPINE
   redistribute connected route-map RM_REDISTRIBUTE_CONNECTED
!
end
Leaf-2#


```
</details>

<details>
<summary> Leaf-3 </summary>
  
```

Leaf-3#
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
service routing protocols model ribd
!
hostname Leaf-3
!
spanning-tree mode mstp
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
   description to Server-3
   no switchport
   ip address 10.4.3.1/24
!
interface Ethernet4
   description to Server-4
   no switchport
   ip address 10.4.4.1/24
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
ip routing
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
   neighbor SPINE peer group
   neighbor SPINE remote-as 65000
   neighbor SPINE bfd
   neighbor SPINE password 7 sWIT5buW1FYPyTvv4vVCmQ==
   neighbor SPINE send-community
   neighbor 10.2.1.4 peer group SPINE
   neighbor 10.2.2.4 peer group SPINE
   redistribute connected route-map RM_REDISTRIBUTE_CONNECTED
!
end
Leaf-3#

```
</details>


---
#### 4. Проверка наличия IP связанности

##### 4.1 Проверка установления соседства BGP со всеми Leaf коммутаторами на примере Spine-1 коммутатора:
<details>
<summary> Spine-1#sh ip bgp summary </summary>
  
```
Spine-1#sh ip bgp summary
BGP summary information for VRF default
Router identifier 10.0.1.0, local AS number 65000
Neighbor Status Codes: m - Under maintenance
  Neighbor         V  AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  10.2.1.1         4  65001            633       632    0    0 00:31:19 Estab   2      2
  10.2.1.3         4  65002            633       632    0    0 00:31:19 Estab   2      2
  10.2.1.5         4  65003            633       632    0    0 00:31:19 Estab   2      2
Spine-1#

```
</details>

<details>
<summary> Spine-1#sh ip bgp </summary>
  
```
Spine-1#sh ip bgp
BGP routing table information for VRF default
Router identifier 10.0.1.0, local AS number 65000
Route status codes: * - valid, > - active, # - not installed, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

         Network                Next Hop            Metric  LocPref Weight  Path
 * >     10.0.0.1/32            10.2.1.1              0       100     0       65001 i
 * >     10.0.0.2/32            10.2.1.3              0       100     0       65002 i
 * >     10.0.0.3/32            10.2.1.5              0       100     0       65003 i
 * >     10.1.0.1/32            10.2.1.1              0       100     0       65001 i
 * >     10.1.0.2/32            10.2.1.3              0       100     0       65002 i
 * >     10.1.0.3/32            10.2.1.5              0       100     0       65003 i
Spine-1#

```

</details>


##### 4.2 Проверка таблицы маршрутизации на примере Spine-1 коммутатора. 
Из таблицы маршрутизации видно, что подсети интерфейсов Lo0 и Lo1 коммутаторов Leaf-1, Leaf-2 и Leaf-3 локального PoD присутствуют и получены через L1 уровень отношений, а подсети интерфейсов Lo0 и Lo1 коммутаторов Leaf-4, Leaf-5 и Leaf-6 второго PoD присутствуют и получены через L2 уровень отношений и доступны через ECMP маршруты через оба SuperSpine коммутатора:

<details>
<summary> Spine-1#sh ip route bgp </summary>

```
Spine-1#sh ip route bgp

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

 B E      10.0.0.1/32 [200/0] via 10.2.1.1, Ethernet1
 B E      10.0.0.2/32 [200/0] via 10.2.1.3, Ethernet2
 B E      10.0.0.3/32 [200/0] via 10.2.1.5, Ethernet3
 B E      10.1.0.1/32 [200/0] via 10.2.1.1, Ethernet1
 B E      10.1.0.2/32 [200/0] via 10.2.1.3, Ethernet2
 B E      10.1.0.3/32 [200/0] via 10.2.1.5, Ethernet3

Spine-1#sh ip route

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

Gateway of last resort is not set

 B E      10.0.0.1/32 [200/0] via 10.2.1.1, Ethernet1
 B E      10.0.0.2/32 [200/0] via 10.2.1.3, Ethernet2
 B E      10.0.0.3/32 [200/0] via 10.2.1.5, Ethernet3
 C        10.0.1.0/32 is directly connected, Loopback0
 B E      10.1.0.1/32 [200/0] via 10.2.1.1, Ethernet1
 B E      10.1.0.2/32 [200/0] via 10.2.1.3, Ethernet2
 B E      10.1.0.3/32 [200/0] via 10.2.1.5, Ethernet3
 C        10.1.1.0/32 is directly connected, Loopback1
 C        10.2.1.0/31 is directly connected, Ethernet1
 C        10.2.1.2/31 is directly connected, Ethernet2
 C        10.2.1.4/31 is directly connected, Ethernet3

Spine-1#


```
</details>

##### 4.3 Проверка таблицы маршрутизации на примере Leaf-1 коммутатора. 
Из таблицы маршрутизации видно, что для подсетей интерфейсов Lo0 и Lo1 коммутаторов Leaf-2 и Leaf-3 присутствуют ECMP маршруты через оба Spine коммутатора.
 
<details>
<summary> Leaf-1# show ip route  </summary>
  
```
Leaf-1#sh ip route

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

Gateway of last resort is not set

 C        10.0.0.1/32 is directly connected, Loopback0
 B E      10.0.0.2/32 [200/0] via 10.2.1.0, Ethernet1
                              via 10.2.2.0, Ethernet2
 B E      10.0.0.3/32 [200/0] via 10.2.1.0, Ethernet1
                              via 10.2.2.0, Ethernet2
 C        10.1.0.1/32 is directly connected, Loopback1
 B E      10.1.0.2/32 [200/0] via 10.2.1.0, Ethernet1
                              via 10.2.2.0, Ethernet2
 B E      10.1.0.3/32 [200/0] via 10.2.1.0, Ethernet1
                              via 10.2.2.0, Ethernet2
 C        10.2.1.0/31 is directly connected, Ethernet1
 C        10.2.2.0/31 is directly connected, Ethernet2
 C        10.4.1.0/24 is directly connected, Ethernet3
  
```
</details>

<details>
<summary> Leaf-1# show ip route  </summary>
  
```

Leaf-1#sh ip route bgp

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

 B E      10.0.0.2/32 [200/0] via 10.2.1.0, Ethernet1
                              via 10.2.2.0, Ethernet2
 B E      10.0.0.3/32 [200/0] via 10.2.1.0, Ethernet1
                              via 10.2.2.0, Ethernet2
 B E      10.1.0.2/32 [200/0] via 10.2.1.0, Ethernet1
                              via 10.2.2.0, Ethernet2
 B E      10.1.0.3/32 [200/0] via 10.2.1.0, Ethernet1
                              via 10.2.2.0, Ethernet2

Leaf-1#


```
</details>


##### 4.4 Проверка доступности по ICMP всех Lo1 интерфейсов между Leaf на примере Leaf-1 коммутатора:

<details>
<summary> Leaf-1# ping 10.1.0.X source loopback 1 </summary>
  
```
Leaf-1#ping 10.1.0.2 source loopback 1
PING 10.1.0.2 (10.1.0.2) from 10.1.0.1 : 72(100) bytes of data.
80 bytes from 10.1.0.2: icmp_seq=1 ttl=63 time=23.3 ms
80 bytes from 10.1.0.2: icmp_seq=2 ttl=63 time=18.5 ms
80 bytes from 10.1.0.2: icmp_seq=3 ttl=63 time=13.5 ms
80 bytes from 10.1.0.2: icmp_seq=4 ttl=63 time=13.4 ms
80 bytes from 10.1.0.2: icmp_seq=5 ttl=63 time=14.2 ms

--- 10.1.0.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 78ms
rtt min/avg/max/mdev = 13.430/16.612/23.304/3.840 ms, pipe 2, ipg/ewma 19.745/19.757 ms
Leaf-1#ping 10.0.0.2 source loopback 1
PING 10.0.0.2 (10.0.0.2) from 10.1.0.1 : 72(100) bytes of data.
80 bytes from 10.0.0.2: icmp_seq=1 ttl=63 time=20.3 ms
80 bytes from 10.0.0.2: icmp_seq=2 ttl=63 time=16.5 ms
80 bytes from 10.0.0.2: icmp_seq=3 ttl=63 time=12.2 ms
80 bytes from 10.0.0.2: icmp_seq=4 ttl=63 time=15.8 ms
80 bytes from 10.0.0.2: icmp_seq=5 ttl=63 time=16.5 ms

--- 10.0.0.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 70ms
rtt min/avg/max/mdev = 12.217/16.307/20.367/2.593 ms, pipe 2, ipg/ewma 17.667/18.298 ms
Leaf-1#ping 10.1.0.3 source loopback 1
PING 10.1.0.3 (10.1.0.3) from 10.1.0.1 : 72(100) bytes of data.
80 bytes from 10.1.0.3: icmp_seq=1 ttl=63 time=25.2 ms
80 bytes from 10.1.0.3: icmp_seq=2 ttl=63 time=18.5 ms
80 bytes from 10.1.0.3: icmp_seq=3 ttl=63 time=15.4 ms
80 bytes from 10.1.0.3: icmp_seq=4 ttl=63 time=13.6 ms
80 bytes from 10.1.0.3: icmp_seq=5 ttl=63 time=14.8 ms

--- 10.1.0.3 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 81ms
rtt min/avg/max/mdev = 13.684/17.564/25.281/4.183 ms, pipe 2, ipg/ewma 20.379/21.204 ms
Leaf-1#ping 10.0.0.3 source loopback 1
PING 10.0.0.3 (10.0.0.3) from 10.1.0.1 : 72(100) bytes of data.
80 bytes from 10.0.0.3: icmp_seq=1 ttl=63 time=22.1 ms
80 bytes from 10.0.0.3: icmp_seq=2 ttl=63 time=20.5 ms
80 bytes from 10.0.0.3: icmp_seq=3 ttl=63 time=18.9 ms
80 bytes from 10.0.0.3: icmp_seq=4 ttl=63 time=17.0 ms
80 bytes from 10.0.0.3: icmp_seq=5 ttl=63 time=15.4 ms

--- 10.0.0.3 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 64ms
rtt min/avg/max/mdev = 15.468/18.834/22.157/2.398 ms, pipe 3, ipg/ewma 16.076/20.319 ms
Leaf-1#


```
</details>

##### 4.5 Проверка BFD на примере Spine-1 коммутатора:
 
 <details>
<summary> Spine-1#show bfd peers </summary>
  
```
Spine-1#show bfd peers
VRF name: default
-----------------
DstAddr       MyDisc    YourDisc  Interface/Transport    Type           LastUp
--------- ----------- ----------- -------------------- ------- ----------------
10.2.1.1  3240413054  3923597260        Ethernet1(15)  normal   09/08/25 19:14
10.2.1.3  1556443711  2346482828        Ethernet2(16)  normal   09/08/25 19:14
10.2.1.5  1335873956   104982505        Ethernet3(17)  normal   09/08/25 19:14

   LastDown            LastDiag    State
-------------- ------------------- -----
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up

```
</details>

<details>
<summary> Spine-1#show ip bgp neighbors bfd </summary>
  
```
Spine-1#show ip bgp neighbors bfd
BGP BFD Neighbor Table
Flags: U - BFD is enabled for BGP neighbor and BFD session state is UP
       I - BFD is enabled for BGP neighbor and BFD session state is INIT
       D - BFD is enabled for BGP neighbor and BFD session state is DOWN
       N - BFD is not enabled for BGP neighbor
Neighbor           Interface          Up/Down    State       Flags
10.2.1.1           Ethernet1          00:43:04   Established U
10.2.1.3           Ethernet2          00:43:04   Established U
10.2.1.5           Ethernet3          00:43:04   Established U
Spine-1#

```
</details>







