## ДЗ №5
VxLAN. L2 VNI
***
### Цель:
Настроить Overlay на основе VxLAN EVPN для L2 связанности между клиентами.
#### Задачи:
1. Настроите BGP peering между Leaf и Spine в AF l2vpn evpn.
2. Настроите связанность между клиентами в первой зоне и убедитесь в её наличии
3. Зафиксируете в документации - план работы, адресное пространство, схему сети, конфигурацию устройств
***
### Результаты выполнения:
---
В качестве Underlay будет использоваться eBGP, настроенный в ДЗ №4 [lab4](https://github.com/egorvshch/DC-networks-design/tree/main/lab4), кратко:
- Spine-1 и Spine-2 находятся в AS65000;
- Leaf-1, Leaf-2 и Leaf-3 находятся в AS65001, AS65002 и AS65003 соответственно.
- L2-связность будет настраиваться между хостами в VLAN 10 (между Host-1 (Leaf-1) и Host-3 (Leaf-3)) и VLAN 20 (между Host-2 (Leaf-2) и Host-4 (Leaf-3))

#### 1. Схема сети:
   
![](https://github.com/egorvshch/DC-networks-design/blob/main/lab5/schema_dc_net.JPG)
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
| Host-1	| eth0	| 10.4.10.10	| /24 | Vlan 10 |
| Host-2	| eth0	| 10.4.20.20	| /24 | Vlan 20 |
| Host-3	| eth0	| 10.4.10.30	| /24 | Vlan 10 |
| Host-4	| eth0	| 10.4.20.40	| /24 | Vlan 20 |

</details>

---

#### 2.1. Предварительная донастройка коммутаторов SPINE:

 В ДЗ №4 [lab4](https://github.com/egorvshch/DC-networks-design/tree/main/lab4) Loopback интерфесы на SPINE не были анонсированы, поэтому предварительно добавляем анонс подсетей с помощью route-map RM_REDISTRIBUTE_CONNECTED, а также включим ECMP на SPINE'ах:
```
ip prefix-list DIRECT_CONNECTED
   seq 10 permit 10.0.2.0/32
   seq 20 permit 10.1.2.0/32
!
route-map RM_REDISTRIBUTE_CONNECTED permit 10
   match ip address prefix-list DIRECT_CONNECTED
!
router bgp 65000
 maximum-paths 2 ecmp 2
 redistribute connected route-map RM_REDISTRIBUTE_CONNECTED
```
#### 2.2. Предварительная донастройка коммутаторов LEAF:
- настройка VLAN и интерфейсов в сторону Host'ов,

**Leaf-1**
```

vlan 10
name service10
!
interface Ethernet3
   description Host-1
   switchport trunk allowed vlan 10
   switchport mode trunk
```
**Leaf-2**
```
vlan 20
name service20
!
interface Ethernet3
   description Host-2
   switchport trunk allowed vlan 20
   switchport mode trunk
```
**Leaf-3**
```
vlan 10
name service10
vlan 20
name service20
!
interface Ethernet3
   description Host-3
   no ip address
   switchport
   switchport trunk allowed vlan 10
   switchport mode trunk
interface Ethernet4
   description Host-4
   no ip address
   switchport
   switchport trunk allowed vlan 20
   switchport mode trunk
```

#### 3. Конфигурация оборудования (Arista):

***3.1*** Для работы EVPN на всех коммутаторах Arista предварительно требуется активировать EVPN Capability командой ниже и перезагрузить коммутаторы:

```
service routing protocols model multi-agent
```

***3.2 Настройка BGP EVPN Overlays на коммутаторах SPINE:***

  Соседство по протоколу eBGP EVPN будет устанавливаться через Loopback0 интерфесы, поэтому требуется указать ```ebgp-multihop```. Поскольку это eBGP, то требуется не забыть включить ```next-hop-unchanged```, чтобы next-hop адрес до VTEP не изменялся. Не забываем активировать соседство в AF EVPN. 
  Итого добавляем следующую конфигурацию на обоих SPINE коммутаторах:

```
router bgp 65000
 neighbor EVPN peer group
 neighbor EVPN next-hop-unchanged
 neighbor EVPN update-source Loopback0
 neighbor EVPN ebgp-multihop 3
 neighbor EVPN send-community extended
 bgp listen range 10.0.0.0/29 peer-group EVPN peer-filter EVPN
 !
 address-family evpn
  neighbor EVPN activate
```
***3.3 Настройка BGP EVPN Overlays на коммутаторах LEAF:***

 Конфигурация eBGP на всех LEAF коммутаторах идентична, за исключением адреса локальной AS в ```router bgp [LOCAL_ASN]```.
Общий шаблон конфигурации eBGP EVPN Overlays на LEAF коммутаторах имеет вид:

```
router bgp [LOCAL_ASN]
   neighbor EVPN peer group
   neighbor EVPN remote-as 65000
   neighbor EVPN update-source Loopback0
   neighbor EVPN ebgp-multihop 3
   neighbor EVPN send-community extended
   neighbor 10.0.1.0 peer group EVPN
   neighbor 10.0.2.0 peer group EVPN   
 !
   address-family evpn
     neighbor EVPN activate
```
***3.4 Настройка VXLAN Tunnel Endpoints (VTEP) и L2VXLAN***

В качестве адреса VTEP будет использоваться интерфейс Loopback1. Для BUM трафика будет использоваться Ingress Replication.
Для VLAN10 будем использовать VNI 10010 и для VLAN20 - VNI 10020. RD и Route-Target указываем вручную (формат для RD - ASN:VNI, RT - VLAN:VNI).

**Leaf-1**

```
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan learn-restrict any
!
router bgp 65001
  vlan 10
      rd 65001:10010
      route-target both 10:10010
      redistribute learned
```
**Leaf-2**
```
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 20 vni 10020
   vxlan learn-restrict any
!
router bgp 65002
  vlan 20
      rd 65002:10020
      route-target both 20:10020
      redistribute learned
```
**Leaf-3**
```
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
   vxlan learn-restrict any
!
router bgp 65003
  vlan 10
      rd 65003:10010
      route-target both 10:10010
      redistribute learned
  vlan 20
      rd 65003:10020
      route-target both 20:10020
      redistribute learned
```
###### Полная конфигурация коммутаторов:

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
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan learn-restrict any
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
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 20 vni 10020
   vxlan learn-restrict any
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
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
   vxlan learn-restrict any
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
end
Leaf-3#

```
</details>

---
#### 4. Проверка наличия IP связанности

##### 4.1 Проверка установления соседства по BGP EVPN со всеми LEAF коммутаторами на примере Spine-1 и со всеми SPINE коммутаторами на примере Leaf-3:

**Spine-1**
<details>
<summary> Spine-1# sh bgp evpn summary </summary>
  
```
Spine-1#sh bgp evpn summary
BGP summary information for VRF default
Router identifier 10.0.1.0, local AS number 65000
Neighbor Status Codes: m - Under maintenance
  Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  10.0.0.1 4 65001           3129      3136    0    0 02:12:30 Estab   1      1
  10.0.0.2 4 65002           3152      3136    0    0 02:12:34 Estab   1      1
  10.0.0.3 4 65003           3144      3124    0    0 02:12:34 Estab   2      2
Spine-1#
```
</details>

**Leaf-3**
<details>
<summary> Leaf-3# sh bgp evpn summary </summary>
  
```
Leaf-3#sh bgp evpn summary
BGP summary information for VRF default
Router identifier 10.0.0.3, local AS number 65003
Neighbor Status Codes: m - Under maintenance
  Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  10.0.1.0 4 65000           3240      3259    0    0 02:17:28 Estab   2      2
  10.0.2.0 4 65000           3250      3257    0    0 02:17:12 Estab   2      2
Leaf-3#
```
</details>

##### 4.2 Проверка маршрутов получаемых по BGP EVPN на примерах Spine-1 и Leaf-3:
Видно, что устройства обменялись Type-3 (imet) маршрутами для всех VTEP, однако Type-2 (mac-ip) маршруты пока отсутствуют:

**Spine-1**
<details>
<summary> Spine-1#sh bgp evpn </summary>
  
```
Spine-1#sh bgp evpn
BGP routing table information for VRF default
Router identifier 10.0.1.0, local AS number 65000
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 65001:10010 imet 10.1.0.1
                                 10.1.0.1              -       100     0       65001 i
 * >      RD: 65002:10020 imet 10.1.0.2
                                 10.1.0.2              -       100     0       65002 i
 * >      RD: 65003:10010 imet 10.1.0.3
                                 10.1.0.3              -       100     0       65003 i
 * >      RD: 65003:10020 imet 10.1.0.3
                                 10.1.0.3              -       100     0       65003 i
Spine-1#
```
</details>

**Leaf-3**
<details>
<summary> Leaf-3#sh bgp evpn </summary>
  
```
Leaf-3#sh bgp evpn
BGP routing table information for VRF default
Router identifier 10.0.0.3, local AS number 65003
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
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
Leaf-3#

```
</details>

##### 4.3 Проверка VXLAN конфигурации и таблиц MAC адресов на примере LEAF-3:
Таблица MAC адресов пока пустая.

<details>
<summary> Leaf-3#show interface vxlan1 </summary>

```
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
  Note: All Dynamic VLANs used by VCS are internal VLANs.
        Use 'show vxlan vni' for details.
  Static VRF to VNI mapping is not configured
  Headend replication flood vtep list is:
    10 10.1.0.1
    20 10.1.0.2
  Shared Router MAC is 0000.0000.0000
Leaf-3#

```
</details>

<details>
<summary> Leaf-3#show vxlan vtep </summary>

```
Leaf-3#show vxlan vtep
Remote VTEPS for Vxlan1:

VTEP           Tunnel Type(s)
-------------- --------------
10.1.0.1       flood
10.1.0.2       flood

Total number of remote VTEPS:  2
Leaf-3#

```
</details>

<details>
<summary> Leaf-3#show vxlan vni </summary>

```
Leaf-3#show vxlan vni
VNI to VLAN Mapping for Vxlan1
VNI         VLAN       Source       Interface       802.1Q Tag
----------- ---------- ------------ --------------- ----------
10010       10         static       Ethernet3       10
                                    Vxlan1          10
10020       20         static       Ethernet4       20
                                    Vxlan1          20
```
</details>

<details>
<summary> Leaf-3#show vxlan address-table </summary>

```
Leaf-3#show vxlan address-table
          Vxlan Mac Address Table
----------------------------------------------------------------------

VLAN  Mac Address     Type      Prt  VTEP             Moves   Last Move
----  -----------     ----      ---  ----             -----   ---------
Total Remote Mac Addresses for this criterion: 0
Leaf-3#
```
</details>

##### 4.4 Проверка наличия IP связанности между Host-1 и Host-3 и Host-2 и Host-4:

- проверка с Host-1 до Host-3:
<details>
<summary> Host-1#ping 10.4.10.30 </summary>
  
```
Host-1#ping 10.4.10.30
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.4.10.30, timeout is 2 seconds:
!!!!!
Success rate is 80 percent (4/5), round-trip min/avg/max = 37/78/189 ms
Host-1#

```
</details>

- проверка с Host-2 до Host-4:
<details>
<summary> Host-2#ping 10.4.20.40 </summary>
  
```
Host-2#ping 10.4.20.40
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.4.20.40, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 29/82/199 ms
Host-2#

```
</details>
 
##### 4.5 Повторная роверка маршрутов получаемых по BGP EVPN на примерах Spine-1 и Leaf-3:
Видно, что появились маршруты Type-2 (mac-ip):

**на Spine-1**:
<details>
<summary> Spine-1#sh bgp evpn </summary>
  
```
Spine-1#sh bgp evpn
BGP routing table information for VRF default
Router identifier 10.0.1.0, local AS number 65000
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 65001:10010 mac-ip aabb.cc80.6000
                                 10.1.0.1              -       100     0       65001 i
 * >      RD: 65002:10020 mac-ip aabb.cc80.7000
                                 10.1.0.2              -       100     0       65002 i
 * >      RD: 65003:10010 mac-ip aabb.cc80.8000
                                 10.1.0.3              -       100     0       65003 i
 * >      RD: 65003:10020 mac-ip aabb.cc80.9000
                                 10.1.0.3              -       100     0       65003 i
 * >      RD: 65001:10010 imet 10.1.0.1
                                 10.1.0.1              -       100     0       65001 i
 * >      RD: 65002:10020 imet 10.1.0.2
                                 10.1.0.2              -       100     0       65002 i
 * >      RD: 65003:10010 imet 10.1.0.3
                                 10.1.0.3              -       100     0       65003 i
 * >      RD: 65003:10020 imet 10.1.0.3
                                 10.1.0.3              -       100     0       65003 i
Spine-1#
```
</details>

**на Leaf-3**:
<details>
<summary> Leaf-3#sh bgp evpn </summary>
  
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
 * >Ec    RD: 65002:10020 mac-ip aabb.cc80.7000
                                 10.1.0.2              -       100     0       65000 65002 i
 *  ec    RD: 65002:10020 mac-ip aabb.cc80.7000
                                 10.1.0.2              -       100     0       65000 65002 i
 * >      RD: 65003:10010 mac-ip aabb.cc80.8000
                                 -                     -       -       0       i
 * >      RD: 65003:10020 mac-ip aabb.cc80.9000
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
Leaf-3#
```
</details>

##### 4.6  Повторная проверка VXLAN конфигурации и таблиц MAC адресов на примере LEAF-3:

<details>
<summary> Leaf-3#sh vxlan vtep </summary>

```
Leaf-3#show vxlan vtep
Remote VTEPS for Vxlan1:

VTEP           Tunnel Type(s)
-------------- --------------
10.1.0.1       unicast, flood
10.1.0.2       unicast, flood

Total number of remote VTEPS:  2
```
</details>

<details>
<summary> Leaf-3#show vxlan vni </summary>

```
Leaf-3#show vxlan vni
VNI to VLAN Mapping for Vxlan1
VNI         VLAN       Source       Interface       802.1Q Tag
----------- ---------- ------------ --------------- ----------
10010       10         static       Ethernet3       10
                                    Vxlan1          10
10020       20         static       Ethernet4       20
                                    Vxlan1          20

VNI to dynamic VLAN Mapping for Vxlan1
VNI       VLAN       VRF       Source
--------- ---------- --------- ------------

Leaf-3#

```
</details>

<details>
<summary> Leaf-3#show vxlan address-table </summary>
  
```
Leaf-3#show vxlan address-table
          Vxlan Mac Address Table
----------------------------------------------------------------------

VLAN  Mac Address     Type      Prt  VTEP             Moves   Last Move
----  -----------     ----      ---  ----             -----   ---------
  10  aabb.cc80.6000  EVPN      Vx1  10.1.0.1         1       0:07:17 ago
  20  aabb.cc80.7000  EVPN      Vx1  10.1.0.2         1       0:06:40 ago
Total Remote Mac Addresses for this criterion: 2
Leaf-3#

```
</details>

<details>
<summary> Leaf-3#show  mac address-table </summary>
  
```
Leaf-3#show  mac address-table
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
  10    aabb.cc80.6000    DYNAMIC     Vx1        1       0:07:51 ago
  10    aabb.cc80.8000    DYNAMIC     Et3        1       0:07:52 ago
  20    aabb.cc80.7000    DYNAMIC     Vx1        1       0:07:14 ago
  20    aabb.cc80.9000    DYNAMIC     Et4        1       0:07:14 ago
Total Mac Addresses for this criterion: 4

          Multicast Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       ----        -----
Total Mac Addresses for this criterion: 0
Leaf-3#

```
</details>

##### 4.7 Детальная проверка маршрута MAC адреса Host-1 (aabb.cc80.6000) и Host-2 (aabb.cc80.7000) на Leaf-3:
Из выводов видно соответствующую информацию по RD, RT, AS-Path, тип инкапсуляции, VNI:

**до  Host-1 (aabb.cc80.6000)**
<details>
<summary> Leaf-3#sh bgp evpn route-type mac-ip aabb.cc80.6000 detail </summary>
  
```
Leaf-3#sh bgp evpn route-type mac-ip aabb.cc80.6000 detail
BGP routing table information for VRF default
Router identifier 10.0.0.3, local AS number 65003
BGP routing table entry for mac-ip aabb.cc80.6000, Route Distinguisher: 65001:10010
 Paths: 2 available
  65000 65001
    10.1.0.1 from 10.0.1.0 (10.0.1.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:10:10010 TunnelEncap:tunnelTypeVxlan
      VNI: 10010 ESI: 0000:0000:0000:0000:0000
  65000 65001
    10.1.0.1 from 10.0.2.0 (10.0.2.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:10:10010 TunnelEncap:tunnelTypeVxlan
      VNI: 10010 ESI: 0000:0000:0000:0000:0000
```
</details>

**до Host-2 (aabb.cc80.7000)**
<details>
<summary> Leaf-3#sh bgp evpn route-type mac-ip  aabb.cc80.7000  detail </summary>
  
```
Leaf-3#sh bgp evpn route-type mac-ip  aabb.cc80.7000  detail
BGP routing table information for VRF default
Router identifier 10.0.0.3, local AS number 65003
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
Leaf-3#

```
</details>


