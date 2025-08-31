## ДЗ №3
Underlay. IS-IS
***
### Цель:
Настроить IS-IS для Underlay сети.
#### Задачи:
1. Настроить ISIS в Underlay сети, для IP связанности между всеми сетевыми устройствами;
2. Зафиксировать в документации - план работы, адресное пространство, схему сети, конфигурацию устройств;
3. Убедиться в наличии IP связанности между устройствами в ISIS домене.
***
### Результаты выполнения:
---
В качестве примера будет использована сеть дата-центра, состоящая из двух PoD, соединенных через Super Spin'ы. Для сокращения флуда и размеров LSDB применим следующий вариант реализации зон ISIS:
- PoD-1 (Spine-1 и Spine-2) будет находиться в Area 1 протокола ISIS;
- PoD-2 (Spine-3 и Spine-4) будет находиться в Area 2 протокола ISIS;
- SuperSpine-1 и SuperSpine-2 будут находиться в Area 0 протокола ISIS
- Все Leaf коммутаторы будут строить отношения только L1, Spine коммутаторы - L1/2, SuperSpine - только L2

NET идентификатор (идентификатор сети) будет иметь вид ```49.X00Y.ZZZZ.ZZZZ.ZZZZ.00``` (AFI.Area ID.System ID.Net-Sel),

где

```49``` - "AFI" (Authority and Format Identifier);

```X00Y``` - Area ID, где ```"X"``` - номер дата-центра (в нашем случае будет 0), ```"Y"``` - номер Area ISIS;

```"ZZZZ.ZZZZ.ZZZZ"``` - форматированный IP адрес интерфейса Loopback0: 

- для Spine коммутаторов будет иметь вид: 10.0.Z.0  => 010.000.00Z.000 =>  0100.0000.Z000 (где Z - номер Spine коммутатора);
- для Leaf коммутаторов будет иметь вид: 10.0.0.F  => 010.000.000.00F =>  0100.0000.000F (где F - номер Leaf коммутатора).

```00``` - Net-Selector

#### 1. Схема сети:
   
![](https://github.com/egorvshch/DC-networks-design/blob/main/lab3/net_scheme_isis.JPG)
---

#### 2. IP-адресация оборудования для Underlay сети ЦОД:
<details>
<summary> IP-адресация оборудования </summary>
   
| Device | Interface	| IP Address | Subnet Mask | description | Device | Interface	| IP Address | Subnet Mask | description |       
|:------ |:-----------|:----------:|:-------------:|:-------------|:------ |:-----------|:----------:|:-------------:|:-------------|
| Spine-1	| Lo0	| 10.0.1.0	| /32 | | Spine-3 | Lo0 | 10.0.3.0 | /32 | |
| | Lo1	| 10.1.1.0 | /32 | | | Lo1 | 10.1.3.0 | /32 | |
| | Eth1	| 10.2.1.0 | /31 | to Leaf-1 | | Eth1 | 10.2.3.0 | /31 | to Leaf-4 |
| | Eth2	| 10.2.1.2 | /31 | to Leaf-2 | | Eth2 | 10.2.3.2 | /31 | to Leaf-5 |
| | Eth3	| 10.2.1.4 | /31 | to Leaf-3 | | Eth3 | 10.2.3.4 | /31 | to Leaf-6 |
| | Eth4 | 10.2.5.1 | /31 | to SuperSpine-1 | | Eth4 | 10.2.5.5 | /31 | to SuperSpine-1 |
| | Eth5 | 10.2.6.1 | /31 | to SuperSpine-2 | | Eth5 | 10.2.6.5 | /31 | to SuperSpine-2 |
| Spine-2	| Lo0	| 10.0.2.0	| /32 | | Spine-4 |Lo0 | 10.0.4.0 |/32| |
| | Lo1	| 10.1.2.0 	| /32 | | | Lo1 | 10.1.4.0 | /32 | |
| | Eth1	| 10.2.2.0	| /31 | to Leaf-1 | | Eth1 | 10.2.4.0 | /31 | to Leaf-4 |
| | Eth2	| 10.2.2.2	| /31 | to Leaf-2 | | Eth2 | 10.2.4.2 | /31 | to Leaf-5 |
| | Eth3	| 10.2.2.4	| /31 | to Leaf-3 | | Eth3 | 10.2.4.4 | /31 | to Leaf-6 |
| | Eth4 | 10.2.5.3 | /31 | to SuperSpine-1 | | Eth4 | 10.2.5.7 | /31 | to SuperSpine-1 |
| | Eth5 | 10.2.6.3 | /31 | to SuperSpine-2 | | Eth5 | 10.2.6.7 | /31 | to SuperSpine-2 |
| Leaf-1	| Lo0	| 10.0.0.1	| /32 | | Leaf-4 | Lo0 |10.0.0.4 | /32 | |
| | Lo1	| 10.1.0.1	| /32 | | | Lo1 | 10.1.0.4 | /32 | |
| | Eth1	| 10.2.1.1	| /31 | to Spine-1 | | Eth1 | 10.2.3.1 | /31 | to Spine-3 |
| | Eth2	| 10.2.2.1	| /31 | to Spine-2 | | Eth2 | 10.2.4.1 | /31 | to Spine-4 |
| | Eth3	| 10.4.1.1	| /24 | to Server-1 | | Eth3 | 10.4.5.1 | /24 | to Server-2-1 |
| Leaf-2	| Lo0	| 10.0.0.2	| /32 | | Leaf-5 | Lo0 | 10.0.0.5 | /32 | |
| | Lo1	| 10.1.0.2	| /32 | | | Lo1 | 10.1.0.5 | /32 | |
| | Eth1	| 10.2.1.3	| /31 | to Spine-1 | | Eth1 | 10.2.3.3 | /31 | to Spine-3 |
| | Eth2	| 10.2.2.3	| /31 | to Spine-2 | | Eth2 | 10.2.4.3 | /31 | to Spine-4 |
| | Eth3	| 10.4.2.1	| /24 | to Server-2 | | Eth3 | 10.4.6.1 | /24 | to Server-2-2 |
| Leaf-3	| Lo0	| 10.0.0.3	| /32 | | Leaf-6 | Lo0 | 10.0.0.6 | /32 | |
| | Lo1	| 10.1.0.3	| /32 | | | Lo1 | 10.1.0.6 | /32 | |
| | Eth1	| 10.2.1.5	| /31 | to Spine-1 | | Eth1 | 10.2.3.5 | /31 | to Spine-3 |
| | Eth2	| 10.2.2.5	| /31 | to Spine-2 | | Eth2 | 10.2.4.5 | /31 | to Spine-4 |
| | Eth3	| 10.4.3.1	| /24 | to Server-3 | | Eth3 | 10.4.7.1 | /24  |to Server-2-3 |
| | Eth4	| 10.4.4.1	| /24 | to Server-4 | | Eth4 | 10.4.8.1 | /24 | to Server-2-4 |
| Server-1	| eth0	| 10.4.1.2	| /24 | | Server-2-1 | eth0 | 10.4.5.2 | /24 | |
| Server-2	| eth0	| 10.4.2.2	| /24 | | Server-2-2 | eth0 | 10.4.6.2 | /24 | |
| Server-3	| eth0	| 10.4.3.2	| /24 | | Server-2-3 | eth0 | 10.4.7.2 | /24 | |
| Server-4	| eth0	| 10.4.4.2	| /24 | | | | | |

</details>

---

#### 3. Конфигурация оборудования (Arista):

 Настройка выполнена с учетом рекомендаций:
 
- Использовать P2P network type на интерфейсах
- Использовать BFD
- Для одного pod достаточно всё соединить L1 линками
- Для большего количества pod лучше разбивать на зоны, либо смотреть в сторону dynamic flooding
- Избегать использования redistribute
- Применять минимально необходимую конфигурацию IS-IS
- Настраивать IS-IS adjacency в GRT (не делать underlay внутри vrf)
- Настройка аутентификации

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
   isis enable UNDERLAY
   isis circuit-type level-1
   isis network point-to-point
   isis authentication mode sha key-id 1
   isis authentication key-id 1 algorithm sha-256 key 7 OVyha/EY3VI=
!
interface Ethernet2
   description to Leaf-2
   no switchport
   ip address 10.2.1.2/31
   isis enable UNDERLAY
   isis circuit-type level-1
   isis network point-to-point
   isis authentication mode sha key-id 1
   isis authentication key-id 1 algorithm sha-256 key 7 OVyha/EY3VI=
!
interface Ethernet3
   description to Leaf-3
   no switchport
   ip address 10.2.1.4/31
   isis enable UNDERLAY
   isis circuit-type level-1
   isis network point-to-point
   isis authentication mode sha key-id 1
   isis authentication key-id 1 algorithm sha-256 key 7 OVyha/EY3VI=
!
interface Ethernet4
   description to SuperSpine-1
   no switchport
   ip address 10.2.5.1/31
   isis enable UNDERLAY
   isis circuit-type level-2
   isis network point-to-point
   isis authentication mode sha key-id 1
   isis authentication key-id 1 algorithm sha-256 key 7 OVyha/EY3VI=
!
interface Ethernet5
   description to SuperSpine-2
   no switchport
   ip address 10.2.6.1/31
   isis enable UNDERLAY
   isis circuit-type level-2
   isis network point-to-point
   isis authentication mode sha key-id 1
   isis authentication key-id 1 algorithm sha-256 key 7 OVyha/EY3VI=
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback0
   ip address 10.0.1.0/32
   isis enable UNDERLAY
   isis passive
!
interface Loopback1
   ip address 10.1.1.0/32
   isis enable UNDERLAY
   isis passive
!
interface Management1
!
ip routing
!
router isis UNDERLAY
   net 49.0001.0100.0000.1000.00
   log-adjacency-changes
   !
   address-family ipv4 unicast
      bfd all-interfaces
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
   isis enable UNDERLAY
   isis circuit-type level-1
   isis network point-to-point
   isis authentication mode sha key-id 1
   isis authentication key-id 1 algorithm sha-256 key 7 OVyha/EY3VI=
!
interface Ethernet2
   description to Leaf-2
   no switchport
   ip address 10.2.2.2/31
   isis enable UNDERLAY
   isis circuit-type level-1
   isis network point-to-point
   isis authentication mode sha key-id 1
   isis authentication key-id 1 algorithm sha-256 key 7 OVyha/EY3VI=
!
interface Ethernet3
   description to Leaf-3
   no switchport
   ip address 10.2.2.4/31
   isis enable UNDERLAY
   isis circuit-type level-1
   isis network point-to-point
   isis authentication mode sha key-id 1
   isis authentication key-id 1 algorithm sha-256 key 7 OVyha/EY3VI=
!
interface Ethernet4
   description to SuperSpine-1
   no switchport
   ip address 10.2.5.3/31
   isis enable UNDERLAY
   isis circuit-type level-2
   isis network point-to-point
   isis authentication mode sha key-id 1
   isis authentication key-id 1 algorithm sha-256 key 7 OVyha/EY3VI=
!
interface Ethernet5
   description to SuperSpine-2
   no switchport
   ip address 10.2.6.3/31
   isis enable UNDERLAY
   isis circuit-type level-2
   isis network point-to-point
   isis authentication mode sha key-id 1
   isis authentication key-id 1 algorithm sha-256 key 7 OVyha/EY3VI=
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback0
   ip address 10.0.2.0/32
   isis enable UNDERLAY
   isis passive
!
interface Loopback1
   ip address 10.1.2.0/32
   isis enable UNDERLAY
   isis passive
!
interface Management1
!
ip routing
!
router isis UNDERLAY
   net 49.0001.0100.0000.2000.00
   log-adjacency-changes
   !
   address-family ipv4 unicast
      bfd all-interfaces
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
   isis enable UNDERLAY
   isis circuit-type level-1
   isis network point-to-point
   isis authentication mode sha key-id 1
   isis authentication key-id 1 algorithm sha-256 key 7 OVyha/EY3VI=
!
interface Ethernet2
   description to Spine-2
   no switchport
   ip address 10.2.2.1/31
   isis enable UNDERLAY
   isis circuit-type level-1
   isis network point-to-point
   isis authentication mode sha key-id 1
   isis authentication key-id 1 algorithm sha-256 key 7 OVyha/EY3VI=
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
   isis enable UNDERLAY
   isis passive
!
interface Loopback1
   ip address 10.1.0.1/32
   isis enable UNDERLAY
   isis passive
!
interface Management1
!
ip routing
!
router isis UNDERLAY
   net 49.0001.0100.0000.0001.00
   is-type level-1
   log-adjacency-changes
   !
   address-family ipv4 unicast
      bfd all-interfaces
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
   isis enable UNDERLAY
   isis circuit-type level-1
   isis network point-to-point
   isis authentication mode sha key-id 1
   isis authentication key-id 1 algorithm sha-256 key 7 OVyha/EY3VI=
!
interface Ethernet2
   description to Spine-2
   no switchport
   ip address 10.2.2.3/31
   isis enable UNDERLAY
   isis circuit-type level-1
   isis network point-to-point
   isis authentication mode sha key-id 1
   isis authentication key-id 1 algorithm sha-256 key 7 OVyha/EY3VI=
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
   isis enable UNDERLAY
   isis passive
!
interface Loopback1
   ip address 10.1.0.2/32
   isis enable UNDERLAY
   isis passive
!
interface Management1
!
ip routing
!
router isis UNDERLAY
   net 49.0001.0100.0000.0002.00
   is-type level-1
   log-adjacency-changes
   !
   address-family ipv4 unicast
      bfd all-interfaces
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
   isis enable UNDERLAY
   isis circuit-type level-1
   isis network point-to-point
   isis authentication mode sha key-id 1
   isis authentication key-id 1 algorithm sha-256 key 7 OVyha/EY3VI=
!
interface Ethernet2
   description to Spine-2
   no switchport
   ip address 10.2.2.5/31
   isis enable UNDERLAY
   isis circuit-type level-1
   isis network point-to-point
   isis authentication mode sha key-id 1
   isis authentication key-id 1 algorithm sha-256 key 7 OVyha/EY3VI=
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
   isis enable UNDERLAY
   isis passive
!
interface Loopback1
   ip address 10.1.0.3/32
   isis enable UNDERLAY
   isis passive
!
interface Management1
!
ip routing
!
router isis UNDERLAY
   net 49.0001.0100.0000.0003.00
   is-type level-1
   log-adjacency-changes
   !
   address-family ipv4 unicast
      bfd all-interfaces
!
end
Leaf-3#

```
</details>

<details>
<summary> Spine-3 </summary>
  
```
Spine-3#sh running-config
! Command: show running-config
! device: Spine-3 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model ribd
!
hostname Spine-3
!
spanning-tree mode mstp
!
interface Ethernet1
   description to Leaf-4
   no switchport
   ip address 10.2.3.0/31
   isis enable UNDERLAY
   isis circuit-type level-1
   isis network point-to-point
   isis authentication mode sha key-id 1
   isis authentication key-id 1 algorithm sha-256 key 7 OVyha/EY3VI=
!
interface Ethernet2
   description to Leaf-5
   no switchport
   ip address 10.2.3.2/31
   isis enable UNDERLAY
   isis circuit-type level-1
   isis network point-to-point
   isis authentication mode sha key-id 1
   isis authentication key-id 1 algorithm sha-256 key 7 OVyha/EY3VI=
!
interface Ethernet3
   description to Leaf-6
   no switchport
   ip address 10.2.3.4/31
   isis enable UNDERLAY
   isis circuit-type level-1
   isis network point-to-point
   isis authentication mode sha key-id 1
   isis authentication key-id 1 algorithm sha-256 key 7 OVyha/EY3VI=
!
interface Ethernet4
   description to SuperSpine-1
   no switchport
   ip address 10.2.5.5/31
   isis enable UNDERLAY
   isis circuit-type level-2
   isis network point-to-point
   isis authentication mode sha key-id 1
   isis authentication key-id 1 algorithm sha-256 key 7 OVyha/EY3VI=
!
interface Ethernet5
   description to SuperSpine-2
   no switchport
   ip address 10.2.6.5/31
   isis enable UNDERLAY
   isis circuit-type level-2
   isis network point-to-point
   isis authentication mode sha key-id 1
   isis authentication key-id 1 algorithm sha-256 key 7 OVyha/EY3VI=
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback0
   ip address 10.0.3.0/32
   isis enable UNDERLAY
   isis passive
!
interface Loopback1
   ip address 10.1.3.0/32
   isis enable UNDERLAY
   isis passive
!
interface Management1
!
ip routing
!
router isis UNDERLAY
   net 49.0002.0100.0000.3000.00
   log-adjacency-changes
   !
   address-family ipv4 unicast
      bfd all-interfaces
!
end
Spine-3#

```
</details>

<details>
<summary> Spine-4 </summary>
  
```

Spine-4#sh running-config
! Command: show running-config
! device: Spine-4 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model ribd
!
hostname Spine-4
!
spanning-tree mode mstp
!
interface Ethernet1
   description to Leaf-4
   no switchport
   ip address 10.2.4.0/31
   isis enable UNDERLAY
   isis circuit-type level-1
   isis network point-to-point
   isis authentication mode sha key-id 1
   isis authentication key-id 1 algorithm sha-256 key 7 OVyha/EY3VI=
!
interface Ethernet2
   description to Leaf-5
   no switchport
   ip address 10.2.4.2/31
   isis enable UNDERLAY
   isis circuit-type level-1
   isis network point-to-point
   isis authentication mode sha key-id 1
   isis authentication key-id 1 algorithm sha-256 key 7 OVyha/EY3VI=
!
interface Ethernet3
   description to Leaf-6
   no switchport
   ip address 10.2.4.4/31
   isis enable UNDERLAY
   isis circuit-type level-1
   isis network point-to-point
   isis authentication mode sha key-id 1
   isis authentication key-id 1 algorithm sha-256 key 7 OVyha/EY3VI=
!
interface Ethernet4
   description to SuperSpine-1
   no switchport
   ip address 10.2.5.7/31
   isis enable UNDERLAY
   isis circuit-type level-2
   isis network point-to-point
   isis authentication mode sha key-id 1
   isis authentication key-id 1 algorithm sha-256 key 7 OVyha/EY3VI=
!
interface Ethernet5
   description to SuperSpine-2
   no switchport
   ip address 10.2.6.7/31
   isis enable UNDERLAY
   isis circuit-type level-2
   isis network point-to-point
   isis authentication mode sha key-id 1
   isis authentication key-id 1 algorithm sha-256 key 7 OVyha/EY3VI=
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback0
   ip address 10.0.4.0/32
   isis enable UNDERLAY
   isis passive
!
interface Loopback1
   ip address 10.1.4.0/32
   isis enable UNDERLAY
   isis passive
!
interface Management1
!
ip routing
!
router isis UNDERLAY
   net 49.0002.0100.0000.4000.00
   log-adjacency-changes
   !
   address-family ipv4 unicast
      bfd all-interfaces
!
end
Spine-4#

```
</details>

<details>
<summary> Leaf-4 </summary>
  
```
Leaf-4#sh running-config
! Command: show running-config
! device: Leaf-4 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model ribd
!
hostname Leaf-4
!
spanning-tree mode mstp
!
interface Ethernet1
   description to Spine-3
   no switchport
   ip address 10.2.3.1/31
   isis enable UNDERLAY
   isis circuit-type level-1
   isis network point-to-point
   isis authentication mode sha key-id 1
   isis authentication key-id 1 algorithm sha-256 key 7 OVyha/EY3VI=
!
interface Ethernet2
   description to Spine-4
   no switchport
   ip address 10.2.4.1/31
   isis enable UNDERLAY
   isis circuit-type level-1
   isis network point-to-point
   isis authentication mode sha key-id 1
   isis authentication key-id 1 algorithm sha-256 key 7 OVyha/EY3VI=
!
interface Ethernet3
   description to Server-2-1
   no switchport
   ip address 10.4.5.1/24
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
   ip address 10.0.0.4/32
   isis enable UNDERLAY
   isis passive
!
interface Loopback1
   ip address 10.1.0.4/32
   isis enable UNDERLAY
   isis passive
!
interface Management1
!
ip routing
!
router isis UNDERLAY
   net 49.0002.0100.0000.0004.00
   is-type level-1
   log-adjacency-changes
   !
   address-family ipv4 unicast
      bfd all-interfaces
!
end
Leaf-4#

```
</details>

<details>
<summary> Leaf-5 </summary>
  
```
Leaf-5#sh running-config
! Command: show running-config
! device: Leaf-5 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model ribd
!
hostname Leaf-5
!
spanning-tree mode mstp
!
interface Ethernet1
   description to Spine-3
   no switchport
   ip address 10.2.3.3/31
   isis enable UNDERLAY
   isis circuit-type level-1
   isis network point-to-point
   isis authentication mode sha key-id 1
   isis authentication key-id 1 algorithm sha-256 key 7 OVyha/EY3VI=
!
interface Ethernet2
   description to Spine-4
   no switchport
   ip address 10.2.4.3/31
   isis enable UNDERLAY
   isis circuit-type level-1
   isis network point-to-point
   isis authentication mode sha key-id 1
   isis authentication key-id 1 algorithm sha-256 key 7 OVyha/EY3VI=
!
interface Ethernet3
   description to Server-2-2
   no switchport
   ip address 10.4.6.1/24
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
   ip address 10.0.0.5/32
   isis enable UNDERLAY
   isis passive
!
interface Loopback1
   ip address 10.1.0.5/32
   isis enable UNDERLAY
   isis passive
!
interface Management1
!
ip routing
!
router isis UNDERLAY
   net 49.0002.0100.0000.0005.00
   is-type level-1
   log-adjacency-changes
   !
   address-family ipv4 unicast
      bfd all-interfaces
!
end
Leaf-5#

```
</details>

<details>
<summary> Leaf-6 </summary>
  
```
Leaf-6#sh running-config
! Command: show running-config
! device: Leaf-6 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model ribd
!
hostname Leaf-6
!
spanning-tree mode mstp
!
interface Ethernet1
   description to Spine-3
   no switchport
   ip address 10.2.3.5/31
   isis enable UNDERLAY
   isis circuit-type level-1
   isis network point-to-point
   isis authentication mode sha key-id 1
   isis authentication key-id 1 algorithm sha-256 key 7 OVyha/EY3VI=
!
interface Ethernet2
   description to Spine-4
   no switchport
   ip address 10.2.4.5/31
   isis enable UNDERLAY
   isis circuit-type level-1
   isis network point-to-point
   isis authentication mode sha key-id 1
   isis authentication key-id 1 algorithm sha-256 key 7 OVyha/EY3VI=
!
interface Ethernet3
   description to Server-2-3
   no switchport
   ip address 10.4.7.1/24
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
   ip address 10.0.0.6/32
   isis enable UNDERLAY
   isis passive
!
interface Loopback1
   ip address 10.1.0.6/32
   isis enable UNDERLAY
   isis passive
!
interface Management1
!
ip routing
!
router isis UNDERLAY
   net 49.0002.0100.0000.0006.00
   is-type level-1
   log-adjacency-changes
   !
   address-family ipv4 unicast
      bfd all-interfaces
!
end
Leaf-6#

```
</details>

<details>
<summary> SuperSpine-1 </summary>
  
```
SuperSpine-1#sh running-config
! Command: show running-config
! device: SuperSpine-1 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model ribd
!
hostname SuperSpine-1
!
spanning-tree mode mstp
!
interface Ethernet1
   description to Spine-1
   no switchport
   ip address 10.2.5.0/31
   isis enable UNDERLAY
   isis circuit-type level-2
   isis network point-to-point
   isis authentication mode sha key-id 1
   isis authentication key-id 1 algorithm sha-256 key 7 OVyha/EY3VI=
!
interface Ethernet2
   description to Spine-2
   no switchport
   ip address 10.2.5.2/31
   isis enable UNDERLAY
   isis circuit-type level-2
   isis network point-to-point
   isis authentication mode sha key-id 1
   isis authentication key-id 1 algorithm sha-256 key 7 OVyha/EY3VI=
!
interface Ethernet3
   description to Spine-3
   no switchport
   ip address 10.2.5.4/31
   isis enable UNDERLAY
   isis circuit-type level-2
   isis network point-to-point
   isis authentication mode sha key-id 1
   isis authentication key-id 1 algorithm sha-256 key 7 OVyha/EY3VI=
!
interface Ethernet4
   description to Spine-4
   no switchport
   ip address 10.2.5.6/31
   isis enable UNDERLAY
   isis circuit-type level-2
   isis network point-to-point
   isis authentication mode sha key-id 1
   isis authentication key-id 1 algorithm sha-256 key 7 OVyha/EY3VI=
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
   ip address 10.0.5.0/32
   isis enable UNDERLAY
   isis passive
!
interface Loopback1
   ip address 10.1.5.0/32
   isis enable UNDERLAY
   isis passive
!
interface Management1
!
ip routing
!
router isis UNDERLAY
   net 49.0000.0100.0000.5000.00
   is-type level-2
   log-adjacency-changes
   !
   address-family ipv4 unicast
      bfd all-interfaces
!
end
SuperSpine-1#

```
</details>
<details>
<summary> SuperSpine-2 </summary>
  
```

SuperSpine-2#sh running-config
! Command: show running-config
! device: SuperSpine-2 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model ribd
!
hostname SuperSpine-2
!
spanning-tree mode mstp
!
interface Ethernet1
   description to Spine-1
   no switchport
   ip address 10.2.6.0/31
   isis enable UNDERLAY
   isis circuit-type level-2
   isis network point-to-point
   isis authentication mode sha key-id 1
   isis authentication key-id 1 algorithm sha-256 key 7 OVyha/EY3VI=
!
interface Ethernet2
   description to Spine-2
   no switchport
   ip address 10.2.6.2/31
   isis enable UNDERLAY
   isis circuit-type level-2
   isis network point-to-point
   isis authentication mode sha key-id 1
   isis authentication key-id 1 algorithm sha-256 key 7 OVyha/EY3VI=
!
interface Ethernet3
   description to Spine-3
   no switchport
   ip address 10.2.6.4/31
   isis enable UNDERLAY
   isis circuit-type level-2
   isis network point-to-point
   isis authentication mode sha key-id 1
   isis authentication key-id 1 algorithm sha-256 key 7 OVyha/EY3VI=
!
interface Ethernet4
   description to Spine-4
   no switchport
   ip address 10.2.6.6/31
   isis enable UNDERLAY
   isis circuit-type level-2
   isis network point-to-point
   isis authentication mode sha key-id 1
   isis authentication key-id 1 algorithm sha-256 key 7 OVyha/EY3VI=
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
   ip address 10.0.6.0/32
   isis enable UNDERLAY
   isis passive
!
interface Loopback1
   ip address 10.1.6.0/32
   isis enable UNDERLAY
   isis passive
!
interface Management1
!
ip routing
!
router isis UNDERLAY
   net 49.0000.0100.0000.6000.00
   is-type level-2
   log-adjacency-changes
   !
   address-family ipv4 unicast
      bfd all-interfaces
!
end
SuperSpine-2#

```
</details>
---

#### 4. Проверка наличия IP связанности

##### 4.1 Проверка установления соседства ISIS со всеми Leaf и SuperSpine коммутаторами и LSDB на примере Spine-1 коммутатора:
<details>
<summary> Spine-1#sh isis neighbors </summary>
  
```
Spine-1#sh isis neighbors

Instance  VRF      System Id        Type Interface          SNPA              State Hold time   Circuit Id
UNDERLAY  default  Leaf-1           L1   Ethernet1          P2P               UP    28          0F
UNDERLAY  default  Leaf-2           L1   Ethernet2          P2P               UP    30          0F
UNDERLAY  default  Leaf-3           L1   Ethernet3          P2P               UP    28          0F
UNDERLAY  default  SuperSpine-1     L2   Ethernet4          P2P               UP    26          0D
UNDERLAY  default  SuperSpine-2     L2   Ethernet5          P2P               UP    23          0D

```
</details>

<details>
<summary> Spine-1#sh isis network topology </summary>
  
```
Spine-1#sh isis network topology

IS-IS Instance: UNDERLAY VRF: default
  IS-IS paths to level-1 routers
    System Id        Metric   IA Metric Next-Hop         Interface                SNPA
    Leaf-1           10       0         Leaf-1           Ethernet1                P2P
    Leaf-2           10       0         Leaf-2           Ethernet2                P2P
    Leaf-3           10       0         Leaf-3           Ethernet3                P2P
    Spine-2          20       0         Leaf-1           Ethernet1                P2P
                                        Leaf-2           Ethernet2                P2P
                                        Leaf-3           Ethernet3                P2P
  IS-IS paths to level-2 routers
    System Id        Metric   IA Metric Next-Hop         Interface                SNPA
    Spine-2          20       0         SuperSpine-1     Ethernet4                P2P
                                        SuperSpine-2     Ethernet5                P2P
    Spine-3          20       0         SuperSpine-1     Ethernet4                P2P
                                        SuperSpine-2     Ethernet5                P2P
    Spine-4          20       0         SuperSpine-1     Ethernet4                P2P
                                        SuperSpine-2     Ethernet5                P2P
    SuperSpine-1     10       0         SuperSpine-1     Ethernet4                P2P
    SuperSpine-2     10       0         SuperSpine-2     Ethernet5                P2P
Spine-1#

```
</details>

<details>
<summary> Spine-1#show isis database </summary>
  
```
Spine-1#show isis database

IS-IS Instance: UNDERLAY VRF: default
  IS-IS Level 1 Link State Database
    LSPID                   Seq Num  Cksum  Life Length IS Flags
    Leaf-1.00-00                 33  63482  1098    135 L1 <>
    Leaf-2.00-00                 17  45378   635    135 L1 <>
    Leaf-3.00-00                 19  16036  1069    135 L1 <>
    Spine-1.00-00                23   2573   864    160 L2 <DefaultAtt>
    Spine-2.00-00                22  23711   409    160 L2 <DefaultAtt>
  IS-IS Level 2 Link State Database
    LSPID                   Seq Num  Cksum  Life Length IS Flags
    Spine-1.00-00                25  14852  1030    264 L2 <>
    Spine-2.00-00                22  18907  1081    264 L2 <>
    Spine-3.00-00                24  27508   939    264 L2 <>
    Spine-4.00-00                21  49158   670    264 L2 <>
    SuperSpine-1.00-00           20  51139   783    189 L2 <>
    SuperSpine-2.00-00           17  28927   811    189 L2 <>
Spine-1#
```
</details>

##### 4.2 Проверка таблицы маршрутизации на примере Spine-1 коммутатора. Из таблицы маршрутизации видно, что:
- подсети интерфейсов Lo0 и Lo1 коммутаторов Leaf-1, Leaf-2 и Leaf-3 локального PoD присутствуют и получены через L1 уровень отношений, а подсети интерфейсов Lo0 и Lo1 коммутаторов Leaf-4, Leaf-5 и Leaf-6 второго PoD присутствуют и получены через L2 уровень отношений и доступны через ECMP маршруты через оба SuperSpine коммутатора:

<details>
<summary> Spine-1#sh ip route </summary>
  
```
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

 I L1     10.0.0.1/32 [115/20] via 10.2.1.1, Ethernet1
 I L1     10.0.0.2/32 [115/20] via 10.2.1.3, Ethernet2
 I L1     10.0.0.3/32 [115/20] via 10.2.1.5, Ethernet3
 I L2     10.0.0.4/32 [115/40] via 10.2.5.0, Ethernet4
                               via 10.2.6.0, Ethernet5
 I L2     10.0.0.5/32 [115/40] via 10.2.5.0, Ethernet4
                               via 10.2.6.0, Ethernet5
 I L2     10.0.0.6/32 [115/40] via 10.2.5.0, Ethernet4
                               via 10.2.6.0, Ethernet5
 C        10.0.1.0/32 is directly connected, Loopback0
 I L1     10.0.2.0/32 [115/30] via 10.2.1.1, Ethernet1
                               via 10.2.1.3, Ethernet2
                               via 10.2.1.5, Ethernet3
 I L2     10.0.3.0/32 [115/30] via 10.2.5.0, Ethernet4
                               via 10.2.6.0, Ethernet5
 I L2     10.0.4.0/32 [115/30] via 10.2.5.0, Ethernet4
                               via 10.2.6.0, Ethernet5
 I L2     10.0.5.0/32 [115/20] via 10.2.5.0, Ethernet4
 I L2     10.0.6.0/32 [115/20] via 10.2.6.0, Ethernet5
 I L1     10.1.0.1/32 [115/20] via 10.2.1.1, Ethernet1
 I L1     10.1.0.2/32 [115/20] via 10.2.1.3, Ethernet2
 I L1     10.1.0.3/32 [115/20] via 10.2.1.5, Ethernet3
 I L2     10.1.0.4/32 [115/40] via 10.2.5.0, Ethernet4
                               via 10.2.6.0, Ethernet5
 I L2     10.1.0.5/32 [115/40] via 10.2.5.0, Ethernet4
                               via 10.2.6.0, Ethernet5
 I L2     10.1.0.6/32 [115/40] via 10.2.5.0, Ethernet4
                               via 10.2.6.0, Ethernet5
 C        10.1.1.0/32 is directly connected, Loopback1
 I L1     10.1.2.0/32 [115/30] via 10.2.1.1, Ethernet1
                               via 10.2.1.3, Ethernet2
                               via 10.2.1.5, Ethernet3
 I L2     10.1.3.0/32 [115/30] via 10.2.5.0, Ethernet4
                               via 10.2.6.0, Ethernet5
 I L2     10.1.4.0/32 [115/30] via 10.2.5.0, Ethernet4
                               via 10.2.6.0, Ethernet5
 I L2     10.1.5.0/32 [115/20] via 10.2.5.0, Ethernet4
 I L2     10.1.6.0/32 [115/20] via 10.2.6.0, Ethernet5
 C        10.2.1.0/31 is directly connected, Ethernet1
 C        10.2.1.2/31 is directly connected, Ethernet2
 C        10.2.1.4/31 is directly connected, Ethernet3
 I L1     10.2.2.0/31 [115/20] via 10.2.1.1, Ethernet1
 I L1     10.2.2.2/31 [115/20] via 10.2.1.3, Ethernet2
 I L1     10.2.2.4/31 [115/20] via 10.2.1.5, Ethernet3
 I L2     10.2.3.0/31 [115/30] via 10.2.5.0, Ethernet4
                               via 10.2.6.0, Ethernet5
 I L2     10.2.3.2/31 [115/30] via 10.2.5.0, Ethernet4
                               via 10.2.6.0, Ethernet5
 I L2     10.2.3.4/31 [115/30] via 10.2.5.0, Ethernet4
                               via 10.2.6.0, Ethernet5
 I L2     10.2.4.0/31 [115/30] via 10.2.5.0, Ethernet4
                               via 10.2.6.0, Ethernet5
 I L2     10.2.4.2/31 [115/30] via 10.2.5.0, Ethernet4
                               via 10.2.6.0, Ethernet5
 I L2     10.2.4.4/31 [115/30] via 10.2.5.0, Ethernet4
                               via 10.2.6.0, Ethernet5
 C        10.2.5.0/31 is directly connected, Ethernet4
 I L2     10.2.5.2/31 [115/20] via 10.2.5.0, Ethernet4
 I L2     10.2.5.4/31 [115/20] via 10.2.5.0, Ethernet4
 I L2     10.2.5.6/31 [115/20] via 10.2.5.0, Ethernet4
 C        10.2.6.0/31 is directly connected, Ethernet5
 I L2     10.2.6.2/31 [115/20] via 10.2.6.0, Ethernet5
 I L2     10.2.6.4/31 [115/20] via 10.2.6.0, Ethernet5
 I L2     10.2.6.6/31 [115/20] via 10.2.6.0, Ethernet5

```
</details>

##### 4.3 Проверка таблицы маршрутизации на примере Leaf-1 коммутатора. Из таблицы маршрутизации видно, что:
 - подсети интерфейсов Lo0 и Lo1 коммутаторов Leaf-2 и Leaf-3 локального PoD присутствуют ECMP маршруты через оба Spine коммутатора локального PoD'а и получены через L1 уровень отношений;
 - подсети интерфейсов Lo0 и Lo1 коммутаторов Leaf-4, Leaf-5 и Leaf-6 в таблице маршрутизации отсутствуют, однако присутствует ECMP маршрут по умолчанию через оба Spine коммутатора. Таким образом, сокращая размер LSDB и Flood домена.

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

Gateway of last resort:
 I L1     0.0.0.0/0 [115/10] via 10.2.1.0, Ethernet1
                             via 10.2.2.0, Ethernet2

 C        10.0.0.1/32 is directly connected, Loopback0
 I L1     10.0.0.2/32 [115/30] via 10.2.1.0, Ethernet1
                               via 10.2.2.0, Ethernet2
 I L1     10.0.0.3/32 [115/30] via 10.2.1.0, Ethernet1
                               via 10.2.2.0, Ethernet2
 I L1     10.0.1.0/32 [115/20] via 10.2.1.0, Ethernet1
 I L1     10.0.2.0/32 [115/20] via 10.2.2.0, Ethernet2
 C        10.1.0.1/32 is directly connected, Loopback1
 I L1     10.1.0.2/32 [115/30] via 10.2.1.0, Ethernet1
                               via 10.2.2.0, Ethernet2
 I L1     10.1.0.3/32 [115/30] via 10.2.1.0, Ethernet1
                               via 10.2.2.0, Ethernet2
 I L1     10.1.1.0/32 [115/20] via 10.2.1.0, Ethernet1
 I L1     10.1.2.0/32 [115/20] via 10.2.2.0, Ethernet2
 C        10.2.1.0/31 is directly connected, Ethernet1
 I L1     10.2.1.2/31 [115/20] via 10.2.1.0, Ethernet1
 I L1     10.2.1.4/31 [115/20] via 10.2.1.0, Ethernet1
 C        10.2.2.0/31 is directly connected, Ethernet2
 I L1     10.2.2.2/31 [115/20] via 10.2.2.0, Ethernet2
 I L1     10.2.2.4/31 [115/20] via 10.2.2.0, Ethernet2
 C        10.4.1.0/24 is directly connected, Ethernet3

Leaf-1#

```
</details>


##### 4.4 Проверка доступности по ICMP всех Lo1 интерфейсов между PoD'ами на примере Leaf-1 коммутатора:

<details>
<summary> Leaf-1# ping 10.1.0.X source loopback 1 </summary>
  
```
LLeaf-1#ping 10.1.0.4 source Loopback 1
PING 10.1.0.4 (10.1.0.4) from 10.1.0.1 : 72(100) bytes of data.
80 bytes from 10.1.0.4: icmp_seq=1 ttl=61 time=134 ms
80 bytes from 10.1.0.4: icmp_seq=2 ttl=61 time=123 ms
80 bytes from 10.1.0.4: icmp_seq=3 ttl=61 time=121 ms
80 bytes from 10.1.0.4: icmp_seq=4 ttl=61 time=113 ms
80 bytes from 10.1.0.4: icmp_seq=5 ttl=61 time=107 ms

--- 10.1.0.4 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 52ms
rtt min/avg/max/mdev = 107.342/119.894/134.111/9.126 ms, pipe 5, ipg/ewma 13.100                                                                                                                                                             /126.367 ms
Leaf-1#ping 10.1.0.5 source Loopback 1
PING 10.1.0.5 (10.1.0.5) from 10.1.0.1 : 72(100) bytes of data.
80 bytes from 10.1.0.5: icmp_seq=1 ttl=61 time=55.8 ms
80 bytes from 10.1.0.5: icmp_seq=2 ttl=61 time=48.4 ms
80 bytes from 10.1.0.5: icmp_seq=3 ttl=61 time=41.9 ms
80 bytes from 10.1.0.5: icmp_seq=4 ttl=61 time=33.2 ms
80 bytes from 10.1.0.5: icmp_seq=5 ttl=61 time=45.1 ms

--- 10.1.0.5 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 53ms
rtt min/avg/max/mdev = 33.267/44.932/55.832/7.434 ms, pipe 5, ipg/ewma 13.488/50                                                                                                                                                             .083 ms
Leaf-1#ping 10.1.0.6 source Loopback 1
PING 10.1.0.6 (10.1.0.6) from 10.1.0.1 : 72(100) bytes of data.
80 bytes from 10.1.0.6: icmp_seq=1 ttl=61 time=64.6 ms
80 bytes from 10.1.0.6: icmp_seq=2 ttl=61 time=63.3 ms
80 bytes from 10.1.0.6: icmp_seq=3 ttl=61 time=54.4 ms
80 bytes from 10.1.0.6: icmp_seq=4 ttl=61 time=45.2 ms
80 bytes from 10.1.0.6: icmp_seq=5 ttl=61 time=43.3 ms

--- 10.1.0.6 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 47ms
rtt min/avg/max/mdev = 43.387/54.200/64.670/8.840 ms, pipe 5, ipg/ewma 11.968/58                                                                                                                                                             .786 ms
Leaf-1#

```
</details>

##### 4.5 Проверка BFD на примере Spine-1 коммутатора:
 
 <details>
<summary> Spine-1#sh bfd peers </summary>
  
```
Spine-1#sh bfd peers
VRF name: default
-----------------
DstAddr       MyDisc    YourDisc  Interface/Transport    Type           LastUp
--------- ----------- ----------- -------------------- ------- ----------------
10.2.1.1  1458043242  1929865312        Ethernet1(15)  normal   08/31/25 16:21
10.2.1.3   569561750  1190005971        Ethernet2(16)  normal   08/31/25 16:21
10.2.1.5  3954966763  1357274803        Ethernet3(17)  normal   08/31/25 16:21
10.2.5.0  1159694629  2508589279        Ethernet4(18)  normal   08/31/25 16:30
10.2.6.0   333140257  1478844231        Ethernet5(19)  normal   08/31/25 16:30

   LastDown            LastDiag    State
-------------- ------------------- -----
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up

```
</details>

<details>
<summary> Spine-1#show bfd peers detail </summary>
  
```
Spine-1#sh bfd peers detail
VRF name: default
-----------------
Peer Addr 10.2.1.1, Intf Ethernet1, Type normal, Role active, State Up
VRF default, LAddr 10.2.1.0, LD/RD 1458043242/1929865312
Session state is Up and not using echo function
Hardware Acceleration: Async Off, Echo Off
Last Up 08/31/25 16:21:26.880
Last Down NA
Last Diag: No Diagnostic
Authentication mode: None
Shared-secret profile: None
TxInt: 300 ms, RxInt: 300 ms, Multiplier: 3
Received RxInt: 300 ms, Received Multiplier: 3
Rx Count: 42397, Rx Interval (ms) min/max/avg: 130/423/271 last: 298 ms ago
Tx Count: 42386, Tx Interval (ms) min/max/avg: 222/338/271 last: 280 ms ago
Detect Time: 900 ms
Sched Delay: 1*TxInt: 35714, 2*TxInt: 6671, 3*TxInt: 0, GT 3*TxInt: 0
Registered protocols: isis
Uptime: 03:11:32.85
Last packet:  Version: 1             - Diagnostic: 0
              State bit: Up          - Demand bit: 0
              Poll bit: 0            - Final bit: 0
              Multiplier: 3          - Length: 24
              My Discr.: 1929865312  - Your Discr.: 1458043242
              Min tx interval: 300   - Min rx interval: 300
              Min Echo interval: 300

Peer Addr 10.2.1.3, Intf Ethernet2, Type normal, Role active, State Up
VRF default, LAddr 10.2.1.2, LD/RD 569561750/1190005971
Session state is Up and not using echo function
Hardware Acceleration: Async Off, Echo Off
Last Up 08/31/25 16:21:21.385
Last Down NA
Last Diag: No Diagnostic
Authentication mode: None
Shared-secret profile: None
TxInt: 300 ms, RxInt: 300 ms, Multiplier: 3
Received RxInt: 300 ms, Received Multiplier: 3
Rx Count: 42361, Rx Interval (ms) min/max/avg: 136/409/271 last: 101 ms ago
Tx Count: 42404, Tx Interval (ms) min/max/avg: 221/344/271 last: 256 ms ago
Detect Time: 900 ms
Sched Delay: 1*TxInt: 35645, 2*TxInt: 6758, 3*TxInt: 0, GT 3*TxInt: 0
Registered protocols: isis
Uptime: 03:11:38.39
Last packet:  Version: 1             - Diagnostic: 0
              State bit: Up          - Demand bit: 0
              Poll bit: 0            - Final bit: 0
              Multiplier: 3          - Length: 24
              My Discr.: 1190005971  - Your Discr.: 569561750
              Min tx interval: 300   - Min rx interval: 300
              Min Echo interval: 300

Peer Addr 10.2.1.5, Intf Ethernet3, Type normal, Role active, State Up
VRF default, LAddr 10.2.1.4, LD/RD 3954966763/1357274803
Session state is Up and not using echo function
Hardware Acceleration: Async Off, Echo Off
Last Up 08/31/25 16:21:27.839
Last Down NA
Last Diag: No Diagnostic
Authentication mode: None
Shared-secret profile: None
TxInt: 300 ms, RxInt: 300 ms, Multiplier: 3
Received RxInt: 300 ms, Received Multiplier: 3
Rx Count: 42357, Rx Interval (ms) min/max/avg: 131/438/271 last: 321 ms ago
Tx Count: 42395, Tx Interval (ms) min/max/avg: 221/328/271 last: 216 ms ago
Detect Time: 900 ms
Sched Delay: 1*TxInt: 35709, 2*TxInt: 6685, 3*TxInt: 0, GT 3*TxInt: 0
Registered protocols: isis
Uptime: 03:11:31.94
Last packet:  Version: 1             - Diagnostic: 0
              State bit: Up          - Demand bit: 0
              Poll bit: 0            - Final bit: 0
              Multiplier: 3          - Length: 24
              My Discr.: 1357274803  - Your Discr.: 3954966763
              Min tx interval: 300   - Min rx interval: 300
              Min Echo interval: 300

Peer Addr 10.2.5.0, Intf Ethernet4, Type normal, Role active, State Up
VRF default, LAddr 10.2.5.1, LD/RD 1159694629/2508589279
Session state is Up and not using echo function
Hardware Acceleration: Async Off, Echo Off
Last Up 08/31/25 16:30:19.911
Last Down NA
Last Diag: No Diagnostic
Authentication mode: None
Shared-secret profile: None
TxInt: 300 ms, RxInt: 300 ms, Multiplier: 3
Received RxInt: 300 ms, Received Multiplier: 3
Rx Count: 40478, Rx Interval (ms) min/max/avg: 79/500/270 last: 33 ms ago
Tx Count: 40415, Tx Interval (ms) min/max/avg: 221/329/271 last: 200 ms ago
Detect Time: 900 ms
Sched Delay: 1*TxInt: 33980, 2*TxInt: 6434, 3*TxInt: 0, GT 3*TxInt: 0
Registered protocols: isis
Uptime: 03:02:39.88
Last packet:  Version: 1             - Diagnostic: 0
              State bit: Up          - Demand bit: 0
              Poll bit: 0            - Final bit: 0
              Multiplier: 3          - Length: 24
              My Discr.: 2508589279  - Your Discr.: 1159694629
              Min tx interval: 300   - Min rx interval: 300
              Min Echo interval: 300

Peer Addr 10.2.6.0, Intf Ethernet5, Type normal, Role active, State Up
VRF default, LAddr 10.2.6.1, LD/RD 333140257/1478844231
Session state is Up and not using echo function
Hardware Acceleration: Async Off, Echo Off
Last Up 08/31/25 16:30:24.913
Last Down NA
Last Diag: No Diagnostic
Authentication mode: None
Shared-secret profile: None
TxInt: 300 ms, RxInt: 300 ms, Multiplier: 3
Received RxInt: 300 ms, Received Multiplier: 3
Rx Count: 40370, Rx Interval (ms) min/max/avg: 120/414/271 last: 166 ms ago
Tx Count: 40417, Tx Interval (ms) min/max/avg: 222/329/271 last: 36 ms ago
Detect Time: 900 ms
Sched Delay: 1*TxInt: 33838, 2*TxInt: 6578, 3*TxInt: 0, GT 3*TxInt: 0
Registered protocols: isis
Uptime: 03:02:34.88
Last packet:  Version: 1             - Diagnostic: 0
              State bit: Up          - Demand bit: 0
              Poll bit: 0            - Final bit: 0
              Multiplier: 3          - Length: 24
              My Discr.: 1478844231  - Your Discr.: 333140257
              Min tx interval: 300   - Min rx interval: 300
              Min Echo interval: 300

Spine-1#

```
</details>

