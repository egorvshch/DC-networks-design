## ДЗ №8
VxLAN. Routing.
***
### Цель:
Реализовать передачу суммарных префиксов через EVPN route-type 5.
#### Задачи:
1. Разместите двух "клиентов" в разных VRF в рамках одной фабрики.
2. Настроите маршрутизацию между клиентами через внешнее устройство (граничный роутер\фаерволл\etc)
3. Зафиксируете в документации - план работы, адресное пространство, схему сети, настройки сетевого оборудования
***
## Результаты выполнения:
---
В качестве исходной схемы использовалась схема из ДЗ №6 [lab6](https://github.com/egorvshch/DC-networks-design/tree/main/lab6), которая была немного скорректирована. 
Кратко:
- Underlay построен на eBGP
- Spine-1 и Spine-2 находятся в AS65000;
- Leaf-1, Leaf-2 и Border-Leaf находятся в AS65001, AS65002 и AS65003 соответственно.
- Host-1 и Host-3 находятся в одном vrf BLUE, подсетях 10.4.10.0/24 и 10.4.20.0/24, агрегированный префикс 10.4.0.0/16
- Host-2 и Host-4 находятся в одном vrf RED, подсетях 10.5.20.0/24 и 10.5.30.0/24, агрегированный префикс 10.5.0.0/16
- Route Leaking осуществляется на Border-FW, который находится в AS65199

### 1. Схема сети:
   
![](https://github.com/egorvshch/DC-networks-design/blob/main/lab8/schema_dc_net.JPG)
---

### 2. IP-адресация оборудования для Underlay сети ЦОД:

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
| | Eth3	| 	|  | to Host-1 |
| | Eth4 | 	|  | to Host-2 |
| Leaf-2	| Lo0	| 10.0.0.2	| /32 |
| | Lo1	| 10.1.0.2	| /32 |
| | Eth1	| 10.2.1.3	| /31 | to Spine-1 |
| | Eth2	| 10.2.2.3	| /31 | to Spine-2 |
| | Eth3	| 	|  | to Host-1 |
| | Eth4 | 	|  | to Host-2 |
| Border-Leaf	| Lo0	| 10.0.0.3	| /32 |
| | Lo1	| 10.1.0.3	| /32 |
| | Eth1	| 10.2.1.5	| /31 | to Spine-1 |
| | Eth2	| 10.2.2.5	| /31 | to Spine-2 |
| | Eth3.104 | 10.99.104.1 | /30 | to Border-FW |
| | Eth3.105 | 10.99.105.1 | /30 | to Border-FW |
| Border-Leaf	| Lo0	|  10.99.0.1	| /32 |
| | Lo88	| 8.8.8.8	| /32 |
| | Lo88	| 8.8.4.4	| /32 |
| | Eth1.104 | 10.99.104.2 | /30 | to Border-Leaf |
| | Eth1.105 | 10.99.105.2 | /30 | to Border-Leaf |
| Host-1	| eth0	| 10.4.10.10	| /24 | Vlan 10 / vrf BLUE |
| Host-2	| eth0	| 10.5.20.20	| /24 | Vlan 20 / vrf RED |
| Host-3	| eth0	| 10.4.20.30	| /24 | Vlan 10 / vrf BLUE |
| Host-4	| eth0	| 10.5.30.40	| /24 | Vlan 20 / vrf RED |

</details>

---

### 3. Настройка оборудрования:

#### 3.1 Настройка Spine-1

<details>
<summary> Базовые настройки и настройки интерфейсов на Spine-1 </summary>

```  
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
interface Loopback0
   ip address 10.0.1.0/32
!
interface Loopback1
   ip address 10.1.1.0/32
!
   
```
</details>
<details>
<summary> Настройки маршрутизации BGP, Underlay и Overlay на Spine-1 </summary>

```  
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
   bgp asn notation asdot
   router-id 10.0.1.0
   timers bgp 3 9
   maximum-paths 2 ecmp 2
   bgp listen range 10.0.0.0/24 peer-group EVPN peer-filter EVPN
   bgp listen range 10.2.1.0/24 peer-group LEAF peer-filter LEAFs
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
```
</details>

[Полная конфигурация Spine-1](https://github.com/egorvshch/DC-networks-design/blob/main/lab8/configs/Spine-1)

#### 3.2 Настройка Spine-2

<details>
<summary> Базовые настройки и настройки интерфейсов на Spine-2 </summary>

```  
service routing protocols model multi-agent
!
hostname Spine-2
!
spanning-tree mode mstp
!
interface Ethernet1
   description to Leaf-1-1
   no switchport
   ip address 10.2.2.0/31
!
interface Ethernet2
   description to Leaf-2-1
   no switchport
   ip address 10.2.2.2/31
!
interface Ethernet3
   description to Leaf-3
   no switchport
   ip address 10.2.2.4/31
!
interface Loopback0
   ip address 10.0.2.0/32
!
interface Loopback1
   ip address 10.1.2.0/32
!
   
```
</details>
<details>
<summary> Настройки маршрутизации BGP, Underlay и Overlay на Spine-2 </summary>

```  
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
   bgp asn notation asdot
   router-id 10.0.2.0
   timers bgp 3 9
   maximum-paths 2 ecmp 2
   bgp listen range 10.0.0.0/24 peer-group EVPN peer-filter EVPN
   bgp listen range 10.2.2.0/24 peer-group LEAF peer-filter LEAFs
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
```
</details>

[Полная конфигурация Spine-2](https://github.com/egorvshch/DC-networks-design/blob/main/lab8/configs/Spine-2)

#### 3.3 Настройка Leaf-1

<details>
<summary> Базовые настройки и настройки интерфейсов на Leaf-1 </summary>

```  
service routing protocols model multi-agent
!
hostname Leaf-1
!
vlan 10
   name service-10
!
vlan 20
   name service-20
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
   description Host-2
   switchport trunk allowed vlan 20
   switchport mode trunk
!
interface Loopback0
   ip address 10.0.0.1/32
!
interface Loopback1
   ip address 10.1.0.1/32
!
vrf instance BLUE
!
vrf instance RED
!
interface Vlan10
   vrf BLUE
   ip address virtual 10.4.10.1/24
!
interface Vlan20
   vrf RED
   ip address virtual 10.5.20.1/24
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
   vxlan vrf BLUE vni 10001
   vxlan vrf RED vni 10002
   vxlan learn-restrict any
!
ip virtual-router mac-address 00:00:00:00:00:01
!
```
</details>

<details>
<summary> Настройки маршрутизации BGP, Underlay и Overlay на Leaf-1 </summary>

```  
ip routing
ip routing vrf BLUE
ip routing vrf RED
!
ip prefix-list DIRECT_CONNECTED
   seq 10 permit 10.0.0.1/32
   seq 20 permit 10.1.0.1/32
!
route-map RM_REDISTRIBUTE_CONNECTED permit 10
   match ip address prefix-list DIRECT_CONNECTED
!
router bgp 65001
   bgp asn notation asdot
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
   vlan 20
      rd 65001:10020
      route-target both 20:10020
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
   vrf RED
      rd 10.1.0.1:2
      route-target import evpn 2:10002
      route-target export evpn 2:10002
      redistribute connected
!
```
</details>

[Полная конфигурация Leaf-1](https://github.com/egorvshch/DC-networks-design/blob/main/lab8/configs/Leaf-1)

#### 3.4 Настройка Leaf-2

<details>
<summary> Базовые настройки и настройки интерфейсов на Leaf-2 </summary>

```  
service routing protocols model multi-agent
!
hostname Leaf-2
!
vlan 10
   name service-10
!
vlan 20
   name service-20
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
   description Host-3
   switchport trunk allowed vlan 10
   switchport mode trunk
!
interface Ethernet4
   description Host-4
   switchport trunk allowed vlan 20
   switchport mode trunk
!
interface Loopback0
   ip address 10.0.0.2/32
!
interface Loopback1
   ip address 10.1.0.2/32
!
vrf instance BLUE
!
vrf instance RED
!
interface Vlan10
   vrf BLUE
   ip address virtual 10.4.20.1/24
!
interface Vlan20
   vrf RED
   ip address virtual 10.5.30.1/24
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
   vxlan vrf BLUE vni 10001
   vxlan vrf RED vni 10002
   vxlan learn-restrict any
!
ip virtual-router mac-address 00:00:00:00:00:01
!
   
```
</details>
<details>
<summary> Настройки маршрутизации BGP, Underlay и Overlay на Leaf-2 </summary>

```  
ip routing
ip routing vrf BLUE
ip routing vrf RED
!
ip prefix-list DIRECT_CONNECTED
   seq 10 permit 10.0.0.2/32
   seq 20 permit 10.1.0.2/32
!
route-map RM_REDISTRIBUTE_CONNECTED permit 10
   match ip address prefix-list DIRECT_CONNECTED
!
router bgp 65002
   bgp asn notation asdot
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
   vlan 10
      rd 65002:10010
      route-target both 10:10010
      redistribute learned
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
   vrf RED
      rd 10.1.0.2:2
      route-target import evpn 2:10002
      route-target export evpn 2:10002
      redistribute connected
!
```
</details>

[Полная конфигурация Leaf-2](https://github.com/egorvshch/DC-networks-design/blob/main/lab8/configs/Leaf-2)

#### 3.5 Настройка Border-Leaf

<details>
<summary> Базовые настройки и настройки интерфейсов на Border-Leaf </summary>

```  
service routing protocols model multi-agent
!
hostname Border-Leaf
!
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
   no switchport
!
vrf instance BLUE
!
vrf instance RED
!
interface Ethernet3.104
   encapsulation dot1q vlan 104
   vrf BLUE
   ip address 10.99.104.1/30
!
interface Ethernet3.105
   encapsulation dot1q vlan 105
   vrf RED
   ip address 10.99.105.1/30
!
interface Loopback0
   ip address 10.0.0.3/32
!
interface Loopback1
   ip address 10.1.0.3/32
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vrf BLUE vni 10001
   vxlan vrf RED vni 10002
   vxlan learn-restrict any
!
ip virtual-router mac-address 00:00:00:00:00:01
!
   
```
</details>

<details>
<summary> Настройки маршрутизации BGP, nderlay и Overlay на Border-Leaf </summary>

```  
ip routing
ip routing vrf BLUE
ip routing vrf RED
!
ip prefix-list DIRECT_CONNECTED
   seq 10 permit 10.0.0.3/32
   seq 20 permit 10.1.0.3/32
!
route-map RM_REDISTRIBUTE_CONNECTED permit 10
   match ip address prefix-list DIRECT_CONNECTED
!
router bgp 65003
   bgp asn notation asdot
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
   address-family evpn
      neighbor EVPN activate
   !
   vrf BLUE
      rd 10.1.0.3:1
      route-target import evpn 1:10001
      route-target export evpn 1:10001
      neighbor 10.99.104.2 remote-as 65199
      aggregate-address 10.4.0.0/16 summary-only
   !
   vrf RED
      rd 10.1.0.3:2
      route-target import evpn 2:10002
      route-target export evpn 2:10002
      neighbor 10.99.105.2 remote-as 65199
      aggregate-address 10.5.0.0/16 summary-only
!
```
</details>

[Полная конфигурация Border-Leaf](https://github.com/egorvshch/DC-networks-design/blob/main/lab8/configs/Border-Leaf)

#### 3.6 Настройка Border-FW

<details>
<summary> Базовые настройки и настройки интерфейсов на Border-FW </summary>

```  
hostname Border-FW
!
interface Ethernet1.104
   encapsulation dot1q vlan 104
   ip address 10.99.104.2/30
!
interface Ethernet1.105
   encapsulation dot1q vlan 105
   ip address 10.99.105.2/30
!
interface Loopback0
   ip address 10.99.0.1/32
!
interface Loopback84
   ip address 8.8.4.4/32
!
interface Loopback88
   ip address 8.8.8.8/32
!   
```
</details>
<details>
<summary> Настройки маршрутизации BGP, Underlay и Overlay на Border-FW </summary>

```  
ip routing
!
route-map REPLACE_AS_IN_104 permit 10
   set as-path match all replacement 65003.104
!
route-map REPLACE_AS_IN_105 permit 10
   set as-path match all replacement 65003.105
!
router bgp 65199
   bgp asn notation asdot
   router-id 10.99.0.1
   timers bgp 3 9
   maximum-paths 24
   neighbor 10.99.104.1 remote-as 65003
   neighbor 10.99.104.1 route-map REPLACE_AS_IN_104 in
   neighbor 10.99.104.1 default-originate
   neighbor 10.99.105.1 remote-as 65003
   neighbor 10.99.105.1 route-map REPLACE_AS_IN_105 in
   neighbor 10.99.105.1 default-originate
!
```
</details>

[Полная конфигурация Border-FW](https://github.com/egorvshch/DC-networks-design/blob/main/lab8/configs/Border-FW)

#### 3.7 Настройка Host-1, Host-2, Host-3 и Host-4
конфигурация хостов указана по сылками ниже:

[Полная конфигурация Host-1](https://github.com/egorvshch/DC-networks-design/blob/main/lab8/configs/Host-1)

[Полная конфигурация Host-2](https://github.com/egorvshch/DC-networks-design/blob/main/lab8/configs/Host-2)

[Полная конфигурация Host-3](https://github.com/egorvshch/DC-networks-design/blob/main/lab8/configs/Host-3)

[Полная конфигурация Host-4](https://github.com/egorvshch/DC-networks-design/blob/main/lab8/configs/Host-4)

#### 4.Проверка настроек

<details>
<summary> 4.1 Проверка связности на Host-1 </summary>

```  
Host-1#ping 10.4.20.30
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.4.20.30, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 41/111/187 ms
Host-1#
Host-1#ping 10.5.20.20
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.5.20.20, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 84/163/339 ms
Host-1#ping 10.5.30.40
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.5.30.40, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 71/94/149 ms
Host-1#
Host-1#ping 8.8.8.8
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 8.8.8.8, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 35/55/116 ms
Host-1#ping 8.8.4.4
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 8.8.4.4, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 37/42/48 ms
Host-1#

```
</details>

<details>
<summary> 4.2 Проверка связности на Host-2 </summary>

```  
Host-2#ping 10.4.10.10
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.4.10.10, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 79/81/82 ms
Host-2#
Host-2#ping 10.4.20.30
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.4.20.30, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 79/109/196 ms
Host-2#ping 10.5.30.40
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.5.30.40, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 36/40/46 ms
Host-2#
Host-2#ping 8.8.8.8
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 8.8.8.8, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 35/39/49 ms
Host-2#ping 8.8.4.4
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 8.8.4.4, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 43/45/53 ms
Host-2#
```
</details>

<details>
<summary> 4.3 Проверка маршрутной информации на Leaf-1 (sh ip route, sh ip bgp) </summary>

```
Leaf-1#sh ip route vrf BLUE

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

Gateway of last resort:
 B E      0.0.0.0/0 [200/0] via VTEP 10.1.0.3 VNI 10001 router-mac 50:00:00:15:f4:e8 local-interface Vxlan1

 C        10.4.10.0/24 is directly connected, Vlan10
 B E      10.4.20.30/32 [200/0] via VTEP 10.1.0.2 VNI 10001 router-mac 50:00:00:03:37:66 local-interface Vxlan1
 B E      10.4.20.0/24 [200/0] via VTEP 10.1.0.2 VNI 10001 router-mac 50:00:00:03:37:66 local-interface Vxlan1
 B E      10.5.0.0/16 [200/0] via VTEP 10.1.0.3 VNI 10001 router-mac 50:00:00:15:f4:e8 local-interface Vxlan1

Leaf-1#
Leaf-1#sh ip route vrf RED

VRF: RED
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

Gateway of last resort:
 B E      0.0.0.0/0 [200/0] via VTEP 10.1.0.3 VNI 10002 router-mac 50:00:00:15:f4:e8 local-interface Vxlan1

 B E      10.4.0.0/16 [200/0] via VTEP 10.1.0.3 VNI 10002 router-mac 50:00:00:15:f4:e8 local-interface Vxlan1
 C        10.5.20.0/24 is directly connected, Vlan20
 B E      10.5.30.40/32 [200/0] via VTEP 10.1.0.2 VNI 10002 router-mac 50:00:00:03:37:66 local-interface Vxlan1
 B E      10.5.30.0/24 [200/0] via VTEP 10.1.0.2 VNI 10002 router-mac 50:00:00:03:37:66 local-interface Vxlan1

Leaf-1#
Leaf-1#sh ip bgp vrf BLUE
BGP routing table information for VRF BLUE
Router identifier 10.4.10.1, local AS number 65001
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >Ec    0.0.0.0/0              10.1.0.3              0       -          100     0       65000 65003 65199 ?
 *  ec    0.0.0.0/0              10.1.0.3              0       -          100     0       65000 65003 65199 ?
 * >      10.4.10.0/24           -                     -       -          -       0       i
 * >Ec    10.4.20.0/24           10.1.0.2              0       -          100     0       65000 65002 i
 *  ec    10.4.20.0/24           10.1.0.2              0       -          100     0       65000 65002 i
 * >Ec    10.4.20.30/32          10.1.0.2              0       -          100     0       65000 65002 i
 *  ec    10.4.20.30/32          10.1.0.2              0       -          100     0       65000 65002 i
 * >Ec    10.5.0.0/16            10.1.0.3              0       -          100     0       65000 65003 65199 65003.105 i
 *  ec    10.5.0.0/16            10.1.0.3              0       -          100     0       65000 65003 65199 65003.105 i
Leaf-1#sh ip bgp vrf RED
BGP routing table information for VRF RED
Router identifier 10.5.20.1, local AS number 65001
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >Ec    0.0.0.0/0              10.1.0.3              0       -          100     0       65000 65003 65199 ?
 *  ec    0.0.0.0/0              10.1.0.3              0       -          100     0       65000 65003 65199 ?
 * >Ec    10.4.0.0/16            10.1.0.3              0       -          100     0       65000 65003 65199 65003.104 i
 *  ec    10.4.0.0/16            10.1.0.3              0       -          100     0       65000 65003 65199 65003.104 i
 * >      10.5.20.0/24           -                     -       -          -       0       i
 * >Ec    10.5.30.0/24           10.1.0.2              0       -          100     0       65000 65002 i
 *  ec    10.5.30.0/24           10.1.0.2              0       -          100     0       65000 65002 i
 * >Ec    10.5.30.40/32          10.1.0.2              0       -          100     0       65000 65002 i
 *  ec    10.5.30.40/32          10.1.0.2              0       -          100     0       65000 65002 i
Leaf-1#

```
</details>

<details>
<summary> 4.4 Проверка EVPN маршрутов на Leaf-1 (sh bgp evpn) </summary>

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
 * >Ec    RD: 65002:10020 mac-ip aabb.cc80.7000 10.5.30.40
                                 10.1.0.2              -       100     0       65000 65002 i
 *  ec    RD: 65002:10020 mac-ip aabb.cc80.7000 10.5.30.40
                                 10.1.0.2              -       100     0       65000 65002 i
 * >Ec    RD: 65002:10010 mac-ip aabb.cc80.8000
                                 10.1.0.2              -       100     0       65000 65002 i
 *  ec    RD: 65002:10010 mac-ip aabb.cc80.8000
                                 10.1.0.2              -       100     0       65000 65002 i
 * >Ec    RD: 65002:10010 mac-ip aabb.cc80.8000 10.4.20.30
                                 10.1.0.2              -       100     0       65000 65002 i
 *  ec    RD: 65002:10010 mac-ip aabb.cc80.8000 10.4.20.30
                                 10.1.0.2              -       100     0       65000 65002 i
 * >      RD: 65001:10020 mac-ip aabb.cc80.9000
                                 -                     -       -       0       i
 * >      RD: 65001:10020 mac-ip aabb.cc80.9000 10.5.20.20
                                 -                     -       -       0       i
 * >      RD: 65001:10010 imet 10.1.0.1
                                 -                     -       -       0       i
 * >      RD: 65001:10020 imet 10.1.0.1
                                 -                     -       -       0       i
 * >Ec    RD: 65002:10010 imet 10.1.0.2
                                 10.1.0.2              -       100     0       65000 65002 i
 *  ec    RD: 65002:10010 imet 10.1.0.2
                                 10.1.0.2              -       100     0       65000 65002 i
 * >Ec    RD: 65002:10020 imet 10.1.0.2
                                 10.1.0.2              -       100     0       65000 65002 i
 *  ec    RD: 65002:10020 imet 10.1.0.2
                                 10.1.0.2              -       100     0       65000 65002 i
 * >Ec    RD: 10.1.0.3:1 ip-prefix 0.0.0.0/0
                                 10.1.0.3              -       100     0       65000 65003 65199 ?
 *  ec    RD: 10.1.0.3:1 ip-prefix 0.0.0.0/0
                                 10.1.0.3              -       100     0       65000 65003 65199 ?
 * >Ec    RD: 10.1.0.3:2 ip-prefix 0.0.0.0/0
                                 10.1.0.3              -       100     0       65000 65003 65199 ?
 *  ec    RD: 10.1.0.3:2 ip-prefix 0.0.0.0/0
                                 10.1.0.3              -       100     0       65000 65003 65199 ?
 * >Ec    RD: 10.1.0.3:2 ip-prefix 10.4.0.0/16
                                 10.1.0.3              -       100     0       65000 65003 65199 65003.104 i
 *  ec    RD: 10.1.0.3:2 ip-prefix 10.4.0.0/16
                                 10.1.0.3              -       100     0       65000 65003 65199 65003.104 i
 * >      RD: 10.1.0.1:1 ip-prefix 10.4.10.0/24
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.2:1 ip-prefix 10.4.20.0/24
                                 10.1.0.2              -       100     0       65000 65002 i
 *  ec    RD: 10.1.0.2:1 ip-prefix 10.4.20.0/24
                                 10.1.0.2              -       100     0       65000 65002 i
 * >Ec    RD: 10.1.0.3:1 ip-prefix 10.5.0.0/16
                                 10.1.0.3              -       100     0       65000 65003 65199 65003.105 i
 *  ec    RD: 10.1.0.3:1 ip-prefix 10.5.0.0/16
                                 10.1.0.3              -       100     0       65000 65003 65199 65003.105 i
 * >      RD: 10.1.0.1:2 ip-prefix 10.5.20.0/24
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.2:2 ip-prefix 10.5.30.0/24
                                 10.1.0.2              -       100     0       65000 65002 i
 *  ec    RD: 10.1.0.2:2 ip-prefix 10.5.30.0/24
                                 10.1.0.2              -       100     0       65000 65002 i
Leaf-1#
Leaf-1#show bgp evpn route-type ip-prefix 10.4.0.0/16
BGP routing table information for VRF default
Router identifier 10.0.0.1, local AS number 65001
BGP routing table entry for ip-prefix 10.4.0.0/16, Route Distinguisher: 10.1.0.3:2
 Paths: 2 available
  65000 65003 65199 65003.104 (aggregated by 65003 10.99.104.1)
    10.1.0.3 from 10.0.1.0 (10.0.1.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor, atomic-aggregate
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:15:f4:e8
      VNI: 10002
  65000 65003 65199 65003.104 (aggregated by 65003 10.99.104.1)
    10.1.0.3 from 10.0.2.0 (10.0.2.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor, atomic-aggregate
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:15:f4:e8
      VNI: 10002
Leaf-1#
Leaf-1#
Leaf-1#show bgp evpn route-type ip-prefix 10.5.0.0/16
BGP routing table information for VRF default
Router identifier 10.0.0.1, local AS number 65001
BGP routing table entry for ip-prefix 10.5.0.0/16, Route Distinguisher: 10.1.0.3:1
 Paths: 2 available
  65000 65003 65199 65003.105 (aggregated by 65003 10.99.105.1)
    10.1.0.3 from 10.0.2.0 (10.0.2.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor, atomic-aggregate
      Extended Community: Route-Target-AS:1:10001 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:15:f4:e8
      VNI: 10001
  65000 65003 65199 65003.105 (aggregated by 65003 10.99.105.1)
    10.1.0.3 from 10.0.1.0 (10.0.1.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor, atomic-aggregate
      Extended Community: Route-Target-AS:1:10001 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:15:f4:e8
      VNI: 10001
Leaf-1#

```
</details>

<details>
<summary> 4.5 Проверка маршрутной информации на Leaf-2 (sh ip route, sh ip bgp) </summary>

```
Leaf-2#sh ip route vrf BLUE

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

Gateway of last resort:
 B E      0.0.0.0/0 [200/0] via VTEP 10.1.0.3 VNI 10001 router-mac 50:00:00:15:f4:e8 local-interface Vxlan1

 B E      10.4.10.10/32 [200/0] via VTEP 10.1.0.1 VNI 10001 router-mac 50:00:00:d5:5d:c0 local-interface Vxlan1
 B E      10.4.10.0/24 [200/0] via VTEP 10.1.0.1 VNI 10001 router-mac 50:00:00:d5:5d:c0 local-interface Vxlan1
 C        10.4.20.0/24 is directly connected, Vlan10
 B E      10.5.0.0/16 [200/0] via VTEP 10.1.0.3 VNI 10001 router-mac 50:00:00:15:f4:e8 local-interface Vxlan1

Leaf-2#sh ip route vrf RED

VRF: RED
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

Gateway of last resort:
 B E      0.0.0.0/0 [200/0] via VTEP 10.1.0.3 VNI 10002 router-mac 50:00:00:15:f4:e8 local-interface Vxlan1

 B E      10.4.0.0/16 [200/0] via VTEP 10.1.0.3 VNI 10002 router-mac 50:00:00:15:f4:e8 local-interface Vxlan1
 B E      10.5.20.20/32 [200/0] via VTEP 10.1.0.1 VNI 10002 router-mac 50:00:00:d5:5d:c0 local-interface Vxlan1
 B E      10.5.20.0/24 [200/0] via VTEP 10.1.0.1 VNI 10002 router-mac 50:00:00:d5:5d:c0 local-interface Vxlan1
 C        10.5.30.0/24 is directly connected, Vlan20

Leaf-2#
Leaf-2#sh ip bgp vrf BLUE
BGP routing table information for VRF BLUE
Router identifier 10.4.20.1, local AS number 65002
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >Ec    0.0.0.0/0              10.1.0.3              0       -          100     0       65000 65003 65199 ?
 *  ec    0.0.0.0/0              10.1.0.3              0       -          100     0       65000 65003 65199 ?
 * >Ec    10.4.10.0/24           10.1.0.1              0       -          100     0       65000 65001 i
 *  ec    10.4.10.0/24           10.1.0.1              0       -          100     0       65000 65001 i
 * >Ec    10.4.10.10/32          10.1.0.1              0       -          100     0       65000 65001 i
 *  ec    10.4.10.10/32          10.1.0.1              0       -          100     0       65000 65001 i
 * >      10.4.20.0/24           -                     -       -          -       0       i
 * >Ec    10.5.0.0/16            10.1.0.3              0       -          100     0       65000 65003 65199 65003.105 i
 *  ec    10.5.0.0/16            10.1.0.3              0       -          100     0       65000 65003 65199 65003.105 i
Leaf-2#
Leaf-2#sh ip bgp vrf RED
BGP routing table information for VRF RED
Router identifier 10.5.20.1, local AS number 65002
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >Ec    0.0.0.0/0              10.1.0.3              0       -          100     0       65000 65003 65199 ?
 *  ec    0.0.0.0/0              10.1.0.3              0       -          100     0       65000 65003 65199 ?
 * >Ec    10.4.0.0/16            10.1.0.3              0       -          100     0       65000 65003 65199 65003.104 i
 *  ec    10.4.0.0/16            10.1.0.3              0       -          100     0       65000 65003 65199 65003.104 i
 * >Ec    10.5.20.0/24           10.1.0.1              0       -          100     0       65000 65001 i
 *  ec    10.5.20.0/24           10.1.0.1              0       -          100     0       65000 65001 i
 * >Ec    10.5.20.20/32          10.1.0.1              0       -          100     0       65000 65001 i
 *  ec    10.5.20.20/32          10.1.0.1              0       -          100     0       65000 65001 i
 * >      10.5.30.0/24           -                     -       -          -       0       i
Leaf-2#
```
</details>

<details>
<summary> 4.6 Проверка EVPN маршрутов на Leaf-1 (sh bgp evpn) </summary>

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
 * >      RD: 65002:10020 mac-ip aabb.cc80.7000 10.5.30.40
                                 -                     -       -       0       i
 * >      RD: 65002:10010 mac-ip aabb.cc80.8000
                                 -                     -       -       0       i
 * >      RD: 65002:10010 mac-ip aabb.cc80.8000 10.4.20.30
                                 -                     -       -       0       i
 * >Ec    RD: 65001:10020 mac-ip aabb.cc80.9000
                                 10.1.0.1              -       100     0       65000 65001 i
 *  ec    RD: 65001:10020 mac-ip aabb.cc80.9000
                                 10.1.0.1              -       100     0       65000 65001 i
 * >Ec    RD: 65001:10020 mac-ip aabb.cc80.9000 10.5.20.20
                                 10.1.0.1              -       100     0       65000 65001 i
 *  ec    RD: 65001:10020 mac-ip aabb.cc80.9000 10.5.20.20
                                 10.1.0.1              -       100     0       65000 65001 i
 * >Ec    RD: 65001:10010 imet 10.1.0.1
                                 10.1.0.1              -       100     0       65000 65001 i
 *  ec    RD: 65001:10010 imet 10.1.0.1
                                 10.1.0.1              -       100     0       65000 65001 i
 * >Ec    RD: 65001:10020 imet 10.1.0.1
                                 10.1.0.1              -       100     0       65000 65001 i
 *  ec    RD: 65001:10020 imet 10.1.0.1
                                 10.1.0.1              -       100     0       65000 65001 i
 * >      RD: 65002:10010 imet 10.1.0.2
                                 -                     -       -       0       i
 * >      RD: 65002:10020 imet 10.1.0.2
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.3:1 ip-prefix 0.0.0.0/0
                                 10.1.0.3              -       100     0       65000 65003 65199 ?
 *  ec    RD: 10.1.0.3:1 ip-prefix 0.0.0.0/0
                                 10.1.0.3              -       100     0       65000 65003 65199 ?
 * >Ec    RD: 10.1.0.3:2 ip-prefix 0.0.0.0/0
                                 10.1.0.3              -       100     0       65000 65003 65199 ?
 *  ec    RD: 10.1.0.3:2 ip-prefix 0.0.0.0/0
                                 10.1.0.3              -       100     0       65000 65003 65199 ?
 * >Ec    RD: 10.1.0.3:2 ip-prefix 10.4.0.0/16
                                 10.1.0.3              -       100     0       65000 65003 65199 65003.104 i
 *  ec    RD: 10.1.0.3:2 ip-prefix 10.4.0.0/16
                                 10.1.0.3              -       100     0       65000 65003 65199 65003.104 i
 * >Ec    RD: 10.1.0.1:1 ip-prefix 10.4.10.0/24
                                 10.1.0.1              -       100     0       65000 65001 i
 *  ec    RD: 10.1.0.1:1 ip-prefix 10.4.10.0/24
                                 10.1.0.1              -       100     0       65000 65001 i
 * >      RD: 10.1.0.2:1 ip-prefix 10.4.20.0/24
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.3:1 ip-prefix 10.5.0.0/16
                                 10.1.0.3              -       100     0       65000 65003 65199 65003.105 i
 *  ec    RD: 10.1.0.3:1 ip-prefix 10.5.0.0/16
                                 10.1.0.3              -       100     0       65000 65003 65199 65003.105 i
 * >Ec    RD: 10.1.0.1:2 ip-prefix 10.5.20.0/24
                                 10.1.0.1              -       100     0       65000 65001 i
 *  ec    RD: 10.1.0.1:2 ip-prefix 10.5.20.0/24
                                 10.1.0.1              -       100     0       65000 65001 i
 * >      RD: 10.1.0.2:2 ip-prefix 10.5.30.0/24
                                 -                     -       -       0       i
Leaf-2#
Leaf-2#sh bgp evpn route-type ip-prefix 10.4.0.0/16 detail
BGP routing table information for VRF default
Router identifier 10.0.0.2, local AS number 65002
BGP routing table entry for ip-prefix 10.4.0.0/16, Route Distinguisher: 10.1.0.3:2
 Paths: 2 available
  65000 65003 65199 65003.104 (aggregated by 65003 10.99.104.1)
    10.1.0.3 from 10.0.1.0 (10.0.1.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor, atomic-aggregate
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:15:f4:e8
      VNI: 10002
  65000 65003 65199 65003.104 (aggregated by 65003 10.99.104.1)
    10.1.0.3 from 10.0.2.0 (10.0.2.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor, atomic-aggregate
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:15:f4:e8
      VNI: 10002
Leaf-2#
Leaf-2#
Leaf-2#sh bgp evpn route-type ip-prefix 10.5.0.0/16 detail
BGP routing table information for VRF default
Router identifier 10.0.0.2, local AS number 65002
BGP routing table entry for ip-prefix 10.5.0.0/16, Route Distinguisher: 10.1.0.3:1
 Paths: 2 available
  65000 65003 65199 65003.105 (aggregated by 65003 10.99.105.1)
    10.1.0.3 from 10.0.2.0 (10.0.2.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor, atomic-aggregate
      Extended Community: Route-Target-AS:1:10001 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:15:f4:e8
      VNI: 10001
  65000 65003 65199 65003.105 (aggregated by 65003 10.99.105.1)
    10.1.0.3 from 10.0.1.0 (10.0.1.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor, atomic-aggregate
      Extended Community: Route-Target-AS:1:10001 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:15:f4:e8
      VNI: 10001
Leaf-2#

```
</details>

<details>
<summary> 4.7 Проверка маршрутной информации на Border-Leaf (sh ip route, sh ip bgp) </summary>

```
Border-Leaf#sh ip route vrf BLUE

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

Gateway of last resort:
 B E      0.0.0.0/0 [200/0] via 10.99.104.2, Ethernet3.104

 B E      10.4.10.10/32 [200/0] via VTEP 10.1.0.1 VNI 10001 router-mac 50:00:00:d5:5d:c0 local-interface Vxlan1
 B E      10.4.10.0/24 [200/0] via VTEP 10.1.0.1 VNI 10001 router-mac 50:00:00:d5:5d:c0 local-interface Vxlan1
 B E      10.4.20.30/32 [200/0] via VTEP 10.1.0.2 VNI 10001 router-mac 50:00:00:03:37:66 local-interface Vxlan1
 B E      10.4.20.0/24 [200/0] via VTEP 10.1.0.2 VNI 10001 router-mac 50:00:00:03:37:66 local-interface Vxlan1
 A B      10.4.0.0/16 is directly connected, Null0
 B E      10.5.0.0/16 [200/0] via 10.99.104.2, Ethernet3.104
 C        10.99.104.0/30 is directly connected, Ethernet3.104

Border-Leaf#
Border-Leaf#sh ip route vrf RED

VRF: RED
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

Gateway of last resort:
 B E      0.0.0.0/0 [200/0] via 10.99.105.2, Ethernet3.105

 B E      10.4.0.0/16 [200/0] via 10.99.105.2, Ethernet3.105
 B E      10.5.20.20/32 [200/0] via VTEP 10.1.0.1 VNI 10002 router-mac 50:00:00:d5:5d:c0 local-interface Vxlan1
 B E      10.5.20.0/24 [200/0] via VTEP 10.1.0.1 VNI 10002 router-mac 50:00:00:d5:5d:c0 local-interface Vxlan1
 B E      10.5.30.40/32 [200/0] via VTEP 10.1.0.2 VNI 10002 router-mac 50:00:00:03:37:66 local-interface Vxlan1
 B E      10.5.30.0/24 [200/0] via VTEP 10.1.0.2 VNI 10002 router-mac 50:00:00:03:37:66 local-interface Vxlan1
 A B      10.5.0.0/16 is directly connected, Null0
 C        10.99.105.0/30 is directly connected, Ethernet3.105

Border-Leaf#sh ip bgp vrf BLUE
BGP routing table information for VRF BLUE
Router identifier 10.99.104.1, local AS number 65003
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      0.0.0.0/0              10.99.104.2           0       -          100     0       65199 ?
 * >      10.4.0.0/16            -                     0       -          -       0       65000 i
 *s>Ec    10.4.10.0/24           10.1.0.1              0       -          100     0       65000 65001 i
 *s ec    10.4.10.0/24           10.1.0.1              0       -          100     0       65000 65001 i
 *s>Ec    10.4.10.10/32          10.1.0.1              0       -          100     0       65000 65001 i
 *s ec    10.4.10.10/32          10.1.0.1              0       -          100     0       65000 65001 i
 *s>Ec    10.4.20.0/24           10.1.0.2              0       -          100     0       65000 65002 i
 *s ec    10.4.20.0/24           10.1.0.2              0       -          100     0       65000 65002 i
 *s>Ec    10.4.20.30/32          10.1.0.2              0       -          100     0       65000 65002 i
 *s ec    10.4.20.30/32          10.1.0.2              0       -          100     0       65000 65002 i
 * >      10.5.0.0/16            10.99.104.2           0       -          100     0       65199 65003.105 i
Border-Leaf#
Border-Leaf#sh ip bgp vrf RED
BGP routing table information for VRF RED
Router identifier 10.99.105.1, local AS number 65003
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      0.0.0.0/0              10.99.105.2           0       -          100     0       65199 ?
 * >      10.4.0.0/16            10.99.105.2           0       -          100     0       65199 65003.104 i
 * >      10.5.0.0/16            -                     0       -          -       0       65000 i
 *s>Ec    10.5.20.0/24           10.1.0.1              0       -          100     0       65000 65001 i
 *s ec    10.5.20.0/24           10.1.0.1              0       -          100     0       65000 65001 i
 *s>Ec    10.5.20.20/32          10.1.0.1              0       -          100     0       65000 65001 i
 *s ec    10.5.20.20/32          10.1.0.1              0       -          100     0       65000 65001 i
 *s>Ec    10.5.30.0/24           10.1.0.2              0       -          100     0       65000 65002 i
 *s ec    10.5.30.0/24           10.1.0.2              0       -          100     0       65000 65002 i
 *s>Ec    10.5.30.40/32          10.1.0.2              0       -          100     0       65000 65002 i
 *s ec    10.5.30.40/32          10.1.0.2              0       -          100     0       65000 65002 i
Border-Leaf#

```
</details>

<details>
<summary> 4.8 Проверка EVPN маршрутов на Border-Leaf (sh bgp evpn) </summary>

```
Border-Leaf#sh bgp evpn
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
 * >Ec    RD: 65002:10020 mac-ip aabb.cc80.7000 10.5.30.40
                                 10.1.0.2              -       100     0       65000 65002 i
 *  ec    RD: 65002:10020 mac-ip aabb.cc80.7000 10.5.30.40
                                 10.1.0.2              -       100     0       65000 65002 i
 * >Ec    RD: 65002:10010 mac-ip aabb.cc80.8000
                                 10.1.0.2              -       100     0       65000 65002 i
 *  ec    RD: 65002:10010 mac-ip aabb.cc80.8000
                                 10.1.0.2              -       100     0       65000 65002 i
 * >Ec    RD: 65002:10010 mac-ip aabb.cc80.8000 10.4.20.30
                                 10.1.0.2              -       100     0       65000 65002 i
 *  ec    RD: 65002:10010 mac-ip aabb.cc80.8000 10.4.20.30
                                 10.1.0.2              -       100     0       65000 65002 i
 * >Ec    RD: 65001:10020 mac-ip aabb.cc80.9000
                                 10.1.0.1              -       100     0       65000 65001 i
 *  ec    RD: 65001:10020 mac-ip aabb.cc80.9000
                                 10.1.0.1              -       100     0       65000 65001 i
 * >Ec    RD: 65001:10020 mac-ip aabb.cc80.9000 10.5.20.20
                                 10.1.0.1              -       100     0       65000 65001 i
 *  ec    RD: 65001:10020 mac-ip aabb.cc80.9000 10.5.20.20
                                 10.1.0.1              -       100     0       65000 65001 i
 * >Ec    RD: 65001:10010 imet 10.1.0.1
                                 10.1.0.1              -       100     0       65000 65001 i
 *  ec    RD: 65001:10010 imet 10.1.0.1
                                 10.1.0.1              -       100     0       65000 65001 i
 * >Ec    RD: 65001:10020 imet 10.1.0.1
                                 10.1.0.1              -       100     0       65000 65001 i
 *  ec    RD: 65001:10020 imet 10.1.0.1
                                 10.1.0.1              -       100     0       65000 65001 i
 * >Ec    RD: 65002:10010 imet 10.1.0.2
                                 10.1.0.2              -       100     0       65000 65002 i
 *  ec    RD: 65002:10010 imet 10.1.0.2
                                 10.1.0.2              -       100     0       65000 65002 i
 * >Ec    RD: 65002:10020 imet 10.1.0.2
                                 10.1.0.2              -       100     0       65000 65002 i
 *  ec    RD: 65002:10020 imet 10.1.0.2
                                 10.1.0.2              -       100     0       65000 65002 i
 * >      RD: 10.1.0.3:1 ip-prefix 0.0.0.0/0
                                 -                     -       100     0       65199 ?
 * >      RD: 10.1.0.3:2 ip-prefix 0.0.0.0/0
                                 -                     -       100     0       65199 ?
 * >      RD: 10.1.0.3:1 ip-prefix 10.4.0.0/16
                                 -                     -       -       0       65000 i
 * >      RD: 10.1.0.3:2 ip-prefix 10.4.0.0/16
                                 -                     -       100     0       65199 65003.104 i
 * >Ec    RD: 10.1.0.1:1 ip-prefix 10.4.10.0/24
                                 10.1.0.1              -       100     0       65000 65001 i
 *  ec    RD: 10.1.0.1:1 ip-prefix 10.4.10.0/24
                                 10.1.0.1              -       100     0       65000 65001 i
 * >Ec    RD: 10.1.0.2:1 ip-prefix 10.4.20.0/24
                                 10.1.0.2              -       100     0       65000 65002 i
 *  ec    RD: 10.1.0.2:1 ip-prefix 10.4.20.0/24
                                 10.1.0.2              -       100     0       65000 65002 i
 * >      RD: 10.1.0.3:1 ip-prefix 10.5.0.0/16
                                 -                     -       100     0       65199 65003.105 i
 * >      RD: 10.1.0.3:2 ip-prefix 10.5.0.0/16
                                 -                     -       -       0       65000 i
 * >Ec    RD: 10.1.0.1:2 ip-prefix 10.5.20.0/24
                                 10.1.0.1              -       100     0       65000 65001 i
 *  ec    RD: 10.1.0.1:2 ip-prefix 10.5.20.0/24
                                 10.1.0.1              -       100     0       65000 65001 i
 * >Ec    RD: 10.1.0.2:2 ip-prefix 10.5.30.0/24
                                 10.1.0.2              -       100     0       65000 65002 i
 *  ec    RD: 10.1.0.2:2 ip-prefix 10.5.30.0/24
                                 10.1.0.2              -       100     0       65000 65002 i
Border-Leaf#
Border-Leaf#sh bgp evpn route-type ip-prefix 10.4.0.0/16 detail
BGP routing table information for VRF default
Router identifier 10.0.0.3, local AS number 65003
BGP routing table entry for ip-prefix 10.4.0.0/16, Route Distinguisher: 10.1.0.3:1
 Paths: 1 available
  65000 (aggregated by 65003 10.99.104.1)
    - from - (0.0.0.0)
      Origin IGP, metric -, localpref -, weight 0, tag 0, valid, local, best, atomic-aggregate
      Extended Community: Route-Target-AS:1:10001 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:15:f4:e8
      VNI: 10001
BGP routing table entry for ip-prefix 10.4.0.0/16, Route Distinguisher: 10.1.0.3:2
 Paths: 1 available
  65199 65003.104 (aggregated by 65003 10.99.104.1)
    - from - (0.0.0.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, best, atomic-aggregate
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:15:f4:e8
      VNI: 10002
Border-Leaf#
Border-Leaf#sh bgp evpn route-type ip-prefix 10.5.0.0/16 detail
BGP routing table information for VRF default
Router identifier 10.0.0.3, local AS number 65003
BGP routing table entry for ip-prefix 10.5.0.0/16, Route Distinguisher: 10.1.0.3:1
 Paths: 1 available
  65199 65003.105 (aggregated by 65003 10.99.105.1)
    - from - (0.0.0.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, best, atomic-aggregate
      Extended Community: Route-Target-AS:1:10001 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:15:f4:e8
      VNI: 10001
BGP routing table entry for ip-prefix 10.5.0.0/16, Route Distinguisher: 10.1.0.3:2
 Paths: 1 available
  65000 (aggregated by 65003 10.99.105.1)
    - from - (0.0.0.0)
      Origin IGP, metric -, localpref -, weight 0, tag 0, valid, local, best, atomic-aggregate
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:15:f4:e8
      VNI: 10002
Border-Leaf#

```
</details>
<details>
<summary> 4.9 Проверка маршрутной информации на Border-FW (sh ip route, sh ip bgp) </summary>

```
BBorder-FW#sh ip route

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

 C        8.8.4.4/32 is directly connected, Loopback84
 C        8.8.8.8/32 is directly connected, Loopback88
 B E      10.4.0.0/16 [200/0] via 10.99.104.1, Ethernet1.104
 B E      10.5.0.0/16 [200/0] via 10.99.105.1, Ethernet1.105
 C        10.99.0.1/32 is directly connected, Loopback0
 C        10.99.104.0/30 is directly connected, Ethernet1.104
 C        10.99.105.0/30 is directly connected, Ethernet1.105

Border-FW#
Border-FW#sh ip bgp
BGP routing table information for VRF default
Router identifier 10.99.0.1, local AS number 65199
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.4.0.0/16            10.99.104.1           0       -          100     0       65003.104 i
 * >      10.5.0.0/16            10.99.105.1           0       -          100     0       65003.105 i
Border-FW#
```
</details>

### 5. Итоги:
Из выводов маршрутной информации видно, что поставленные задачи выполнены, цели ДЗ достигнуты:
- клиенты разных vrf имеют связность между собой;
- маршрутизация между клиентами осуществляется через внешнее устройсвто Border-FW;
- обмен маршрутной информации осуществляется с использованием суммарных агрегированных префиксов через EVPN route-type 5.
