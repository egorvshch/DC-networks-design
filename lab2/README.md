### ДЗ №1
Underlay. OSPF
***
#### Цель:
Настроить OSPF для Underlay сети.
***
#### Результаты выполнения:
---
Поскольку в нашей тестовой сети не предполагается больше одного POD'a и количество коммутаторов не значительно, то все коммутаторы будут находиться в одной Backbone Area 0 протокола OSPF.
В качесте RID будут использоваться адреса интерфейсов Lo0.

##### 1. Схема сети:
   
![](https://github.com/egorvshch/DC-networks-design/blob/main/lab2/net_scheme_ospf.jpg)
---

##### 2. IP-адресация оборудования для Underlay сети ЦОД:
<details>
<summary> IP-адресация оборудования </summary>
   
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

##### 3. Конфигурация оборудования (Arista):

<details>
<summary> Spine-1 </summary>
  
```
Spine-1#sh run
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
   ip ospf network point-to-point
   ip ospf authentication message-digest
   ip ospf area 0.0.0.0
   ip ospf message-digest-key 1 sha256 7 doY42AO/Btg=
!
interface Ethernet2
   description to Leaf-2
   no switchport
   ip address 10.2.1.2/31
   ip ospf network point-to-point
   ip ospf authentication message-digest
   ip ospf area 0.0.0.0
   ip ospf message-digest-key 1 sha256 7 8KY28HxgvF0=
!
interface Ethernet3
   description to Leaf-3
   no switchport
   ip address 10.2.1.4/31
   ip ospf network point-to-point
   ip ospf authentication message-digest
   ip ospf area 0.0.0.0
   ip ospf message-digest-key 1 sha256 7 8KY28HxgvF0=
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
   ip address 10.0.1.0/32
   ip ospf area 0.0.0.0
!
interface Loopback1
   ip address 10.1.1.0/32
   ip ospf area 0.0.0.0
!
interface Management1
!
ip routing
!
router ospf 1
   router-id 10.0.1.0
   passive-interface default
   no passive-interface Ethernet1
   no passive-interface Ethernet2
   no passive-interface Ethernet3
   max-lsa 12000
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
   ip ospf network point-to-point
   ip ospf authentication message-digest
   ip ospf area 0.0.0.0
   ip ospf message-digest-key 1 sha256 7 doY42AO/Btg=
!
interface Ethernet2
   description to Leaf-2
   no switchport
   ip address 10.2.2.2/31
   ip ospf network point-to-point
   ip ospf authentication message-digest
   ip ospf area 0.0.0.0
   ip ospf message-digest-key 1 sha256 7 8KY28HxgvF0=
!
interface Ethernet3
   description to Leaf-3
   no switchport
   ip address 10.2.2.4/31
   ip ospf network point-to-point
   ip ospf authentication message-digest
   ip ospf area 0.0.0.0
   ip ospf message-digest-key 1 sha256 7 8KY28HxgvF0=
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
   ip address 10.0.2.0/32
   ip ospf area 0.0.0.0
!
interface Loopback1
   ip address 10.1.2.0/32
   ip ospf area 0.0.0.0
!
interface Management1
!
ip routing
!
router ospf 1
   router-id 10.0.2.0
   passive-interface default
   no passive-interface Ethernet1
   no passive-interface Ethernet2
   no passive-interface Ethernet3
   max-lsa 12000
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
   ip ospf network point-to-point
   ip ospf authentication message-digest
   ip ospf area 0.0.0.0
   ip ospf message-digest-key 1 sha256 7 doY42AO/Btg=
!
interface Ethernet2
   description to Spine-2
   no switchport
   ip address 10.2.2.1/31
   ip ospf network point-to-point
   ip ospf authentication message-digest
   ip ospf area 0.0.0.0
   ip ospf message-digest-key 1 sha256 7 8KY28HxgvF0=
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
   ip ospf area 0.0.0.0
!
interface Loopback1
   ip address 10.1.0.1/32
   ip ospf area 0.0.0.0
!
interface Management1
!
ip routing
!
router ospf 1
   router-id 10.0.0.1
   passive-interface default
   no passive-interface Ethernet1
   no passive-interface Ethernet2
   no passive-interface Ethernet3
   max-lsa 12000
!
end
Leaf-1#

```
</details>

<details>
<summary> Leaf-2 </summary>
  
```
Leaf-2#sh run
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
   ip ospf network point-to-point
   ip ospf authentication message-digest
   ip ospf area 0.0.0.0
   ip ospf message-digest-key 1 sha256 7 doY42AO/Btg=
!
interface Ethernet2
   description to Spine-2
   no switchport
   ip address 10.2.2.3/31
   ip ospf network point-to-point
   ip ospf authentication message-digest
   ip ospf area 0.0.0.0
   ip ospf message-digest-key 1 sha256 7 8KY28HxgvF0=
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
   ip ospf area 0.0.0.0
!
interface Loopback1
   ip address 10.1.0.2/32
   ip ospf area 0.0.0.0
!
interface Management1
!
ip routing
!
router ospf 1
   router-id 10.0.0.2
   passive-interface default
   no passive-interface Ethernet1
   no passive-interface Ethernet2
   no passive-interface Ethernet3
   max-lsa 12000
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
   ip ospf network point-to-point
   ip ospf authentication message-digest
   ip ospf area 0.0.0.0
   ip ospf message-digest-key 1 sha256 7 doY42AO/Btg=
!
interface Ethernet2
   description to Spine-2
   no switchport
   ip address 10.2.2.5/31
   ip ospf network point-to-point
   ip ospf authentication message-digest
   ip ospf area 0.0.0.0
   ip ospf message-digest-key 1 sha256 7 8KY28HxgvF0=
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
   ip ospf area 0.0.0.0
!
interface Loopback1
   ip address 10.1.0.3/32
   ip ospf area 0.0.0.0
!
interface Management1
!
ip routing
!
router ospf 1
   router-id 10.0.0.3
   passive-interface default
   no passive-interface Ethernet1
   no passive-interface Ethernet2
   no passive-interface Ethernet3
   max-lsa 12000
!
end
Leaf-3#

```
</details>

---

#### 4. Проверка наличия IP связанности

 4.1 Проверка установления соседства OSPF на Spine коммутаторах со всеми тремя Leaf коммутаторами:
<details>
<summary> Spine-1# show ip ospf neighbor </summary>
  
```
Spine-1#sh ip ospf neighbor
Neighbor ID     Instance VRF      Pri State                  Dead Time   Address         Interface
10.0.0.1        1        default  0   FULL                   00:00:37    10.2.1.1        Ethernet1
10.0.0.2        1        default  0   FULL                   00:00:33    10.2.1.3        Ethernet2
10.0.0.3        1        default  0   FULL                   00:00:29    10.2.1.5        Ethernet3
Spine-1#

```
</details>

<details>
<summary> Spine-2# ip osfpf neighbor </summary>
  
```
Spine-2#sh ip ospf neighbor
Neighbor ID     Instance VRF      Pri State                  Dead Time   Address         Interface
10.0.0.1        1        default  0   FULL                   00:00:32    10.2.2.1        Ethernet1
10.0.0.2        1        default  0   FULL                   00:00:34    10.2.2.3        Ethernet2
10.0.0.3        1        default  0   FULL                   00:00:32    10.2.2.5        Ethernet3
Spine-2#

```
</details>

 4.2 Проверка таблицы маршрутизации на примере Leaf-1 коммутатора, из которой видно, что интерфейсов Lo0 и lo1 коммутаторов Leaf-2 и Leaf-3 присутствуют ECMP маршруты через оба Spine коммутатора:

<details>
<summary> Leaf-1# show ip route  </summary>
  
```
Leaf-1#show ip route

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
 O        10.0.0.2/32 [110/30] via 10.2.1.0, Ethernet1
                               via 10.2.2.0, Ethernet2
 O        10.0.0.3/32 [110/30] via 10.2.1.0, Ethernet1
                               via 10.2.2.0, Ethernet2
 O        10.0.1.0/32 [110/20] via 10.2.1.0, Ethernet1
 O        10.0.2.0/32 [110/20] via 10.2.2.0, Ethernet2
 C        10.1.0.1/32 is directly connected, Loopback1
 O        10.1.0.2/32 [110/30] via 10.2.1.0, Ethernet1
                               via 10.2.2.0, Ethernet2
 O        10.1.0.3/32 [110/30] via 10.2.1.0, Ethernet1
                               via 10.2.2.0, Ethernet2
 O        10.1.1.0/32 [110/20] via 10.2.1.0, Ethernet1
 O        10.1.2.0/32 [110/20] via 10.2.2.0, Ethernet2
 C        10.2.1.0/31 is directly connected, Ethernet1
 O        10.2.1.2/31 [110/20] via 10.2.1.0, Ethernet1
 O        10.2.1.4/31 [110/20] via 10.2.1.0, Ethernet1
 C        10.2.2.0/31 is directly connected, Ethernet2
 O        10.2.2.2/31 [110/20] via 10.2.2.0, Ethernet2
 O        10.2.2.4/31 [110/20] via 10.2.2.0, Ethernet2
 C        10.4.1.0/24 is directly connected, Ethernet3
Leaf-1#

```
</details>

 4.3 Проверка доступности по ICMP всех Lo интерфейсов на примере Leaf-1 коммутатора:

<details>
<summary> Leaf-1# ping 10.X.X.X source loopback 0 repeat 4 </summary>
  
```
Leaf-1#ping 10.0.0.2 source loopback 0 repeat 4
PING 10.0.0.2 (10.0.0.2) from 10.0.0.1 : 72(100) bytes of data.
80 bytes from 10.0.0.2: icmp_seq=1 ttl=63 time=18.0 ms
80 bytes from 10.0.0.2: icmp_seq=2 ttl=63 time=17.3 ms
80 bytes from 10.0.0.2: icmp_seq=3 ttl=63 time=13.1 ms
80 bytes from 10.0.0.2: icmp_seq=4 ttl=63 time=14.4 ms

--- 10.0.0.2 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 50ms
rtt min/avg/max/mdev = 13.143/15.739/18.064/2.019 ms, pipe 2, ipg/ewma 16.937/17.000 ms
Leaf-1#ping 10.0.0.3 source loopback 0 repeat 4
PING 10.0.0.3 (10.0.0.3) from 10.0.0.1 : 72(100) bytes of data.
80 bytes from 10.0.0.3: icmp_seq=1 ttl=63 time=19.7 ms
80 bytes from 10.0.0.3: icmp_seq=2 ttl=63 time=15.9 ms
80 bytes from 10.0.0.3: icmp_seq=3 ttl=63 time=15.8 ms
80 bytes from 10.0.0.3: icmp_seq=4 ttl=63 time=13.2 ms

--- 10.0.0.3 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 54ms
rtt min/avg/max/mdev = 13.238/16.187/19.704/2.303 ms, pipe 2, ipg/ewma 18.029/18.115 ms
Leaf-1#ping 10.1.0.2 source loopback 0 repeat 4
PING 10.1.0.2 (10.1.0.2) from 10.0.0.1 : 72(100) bytes of data.
80 bytes from 10.1.0.2: icmp_seq=1 ttl=63 time=22.2 ms
80 bytes from 10.1.0.2: icmp_seq=2 ttl=63 time=16.4 ms
80 bytes from 10.1.0.2: icmp_seq=3 ttl=63 time=13.6 ms
80 bytes from 10.1.0.2: icmp_seq=4 ttl=63 time=13.5 ms

--- 10.1.0.2 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 54ms
rtt min/avg/max/mdev = 13.584/16.500/22.255/3.521 ms, pipe 2, ipg/ewma 18.306/19.681 ms
Leaf-1#ping 10.1.0.3 source loopback 0 repeat 4
PING 10.1.0.3 (10.1.0.3) from 10.0.0.1 : 72(100) bytes of data.
80 bytes from 10.1.0.3: icmp_seq=1 ttl=63 time=18.8 ms
80 bytes from 10.1.0.3: icmp_seq=2 ttl=63 time=15.7 ms
80 bytes from 10.1.0.3: icmp_seq=3 ttl=63 time=15.9 ms
80 bytes from 10.1.0.3: icmp_seq=4 ttl=63 time=13.6 ms

--- 10.1.0.3 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 51ms
rtt min/avg/max/mdev = 13.646/16.038/18.857/1.860 ms, pipe 2, ipg/ewma 17.221/17.585 ms
Leaf-1#ping 10.0.1.0 source loopback 0 repeat 4
PING 10.0.1.0 (10.0.1.0) from 10.0.0.1 : 72(100) bytes of data.
80 bytes from 10.0.1.0: icmp_seq=1 ttl=64 time=8.62 ms
80 bytes from 10.0.1.0: icmp_seq=2 ttl=64 time=6.93 ms
80 bytes from 10.0.1.0: icmp_seq=3 ttl=64 time=6.61 ms
80 bytes from 10.0.1.0: icmp_seq=4 ttl=64 time=10.4 ms

--- 10.0.1.0 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 30ms
rtt min/avg/max/mdev = 6.616/8.153/10.432/1.522 ms, ipg/ewma 10.069/8.472 ms
Leaf-1#ping 10.0.2.0 source loopback 0 repeat 4
PING 10.0.2.0 (10.0.2.0) from 10.0.0.1 : 72(100) bytes of data.
80 bytes from 10.0.2.0: icmp_seq=1 ttl=64 time=9.47 ms
80 bytes from 10.0.2.0: icmp_seq=2 ttl=64 time=7.42 ms
80 bytes from 10.0.2.0: icmp_seq=3 ttl=64 time=7.49 ms
80 bytes from 10.0.2.0: icmp_seq=4 ttl=64 time=7.74 ms

--- 10.0.2.0 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 35ms
rtt min/avg/max/mdev = 7.429/8.033/9.471/0.843 ms, ipg/ewma 11.735/8.843 ms
Leaf-1#ping 10.1.1.0 source loopback 0 repeat 4
PING 10.1.1.0 (10.1.1.0) from 10.0.0.1 : 72(100) bytes of data.
80 bytes from 10.1.1.0: icmp_seq=1 ttl=64 time=8.88 ms
80 bytes from 10.1.1.0: icmp_seq=2 ttl=64 time=6.36 ms
80 bytes from 10.1.1.0: icmp_seq=3 ttl=64 time=8.41 ms
80 bytes from 10.1.1.0: icmp_seq=4 ttl=64 time=6.49 ms

--- 10.1.1.0 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 30ms
rtt min/avg/max/mdev = 6.361/7.539/8.886/1.130 ms, ipg/ewma 10.109/8.294 ms
Leaf-1#ping 10.1.2.0 source loopback 0 repeat 4
PING 10.1.2.0 (10.1.2.0) from 10.0.0.1 : 72(100) bytes of data.
80 bytes from 10.1.2.0: icmp_seq=1 ttl=64 time=8.01 ms
80 bytes from 10.1.2.0: icmp_seq=2 ttl=64 time=6.45 ms
80 bytes from 10.1.2.0: icmp_seq=3 ttl=64 time=8.45 ms
80 bytes from 10.1.2.0: icmp_seq=4 ttl=64 time=7.09 ms

--- 10.1.2.0 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 35ms
rtt min/avg/max/mdev = 6.457/7.503/8.451/0.777 ms, ipg/ewma 11.867/7.795 ms
Leaf-1#

```
</details>




