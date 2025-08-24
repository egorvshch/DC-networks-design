### ДЗ №1
Проектирование адресного пространства
***
#### Цель:
1. Собрать схему CLOS;
2. Распределить адресное пространство.
***
#### Результаты выполнения:
---
##### 1. Схема сети:
   
![](https://github.com/egorvshch/DC-networks-design/blob/main/lab1/net_scheme.jpg)
---
##### 2. Распределение адресного пространства для Underlay сети ЦОД:

С учетом предполагаемого масштаба топологии для DC выделяется подсеть 10.0.0.0/13 (порядка 524286 адресов).

2.1. Общая структура распределения подсети:

| Сеть  | Подсеть на DC  | Распределение подсетей |               |                | Назначение | 
|:------------ |:---------------:|:---------------:|:------------:|:---------------:|--------------:|
| 10.0.0.0/8 | 10.0.0.0/13 | 10.0.0.0/14 | 10.0.0.0/15 | 10.0.0.0/16 | for Lo0 |
|                |                |                |                | 10.1.0.0/16 | for Lo1 |
|                |                |                | 10.2.0.0/15 | 10.2.0.0/16 | P2P links |
|                |                |                |                | 10.3.0.0/16 | rezerved |
|                |                | 10.4.0.0/14 | 10.4.0.0/15 | 10.4.0.0/16 | services |
|                |                |                |                | 10.5.0.0/16 | services |
|                |                |                | 10.6.0.0/15 | 10.6.0.0/16 | services |
|                |                |                |                | 10.7.0.0/16 | services |
|                | 10.8.0.0/13 | DC2 |

2.2. Правила распределения адресов:

IP = 10.Dn.Sn.X/XX, где:

| Параметр	| Назначение	| Диапазон | Правило |
|:--------- |:-----------|:---------:|---------|
| Dn |	сервис в зависимости от номера ЦОДа	| 0-7 | Dn для DCN=[8*(N-1)..(8*N-1)], где N – номер ЦОД (DC), N=[1 .. 32] |
| Sn | номер Spine	| 1-255 |
| X	|значение по порядку	| 1-255 |
| XX | требуемая маска	| 8-32 |

Правила распределения внутри ЦОДа:
| Назначение | Правило | Значение | Суммарная подсеть |
|:--------- |:-----------:|:---------:|---------:|
| Lo0 (/16) | (8*(N-1)) | 10.0.0.0/16 | 10.0.0.0/15 |
| Lo1 (/16) | (8*(N-1) + 1) | 10.1.0.0/16 | |
| p2p links (/16) |  (8*(N-1) + 2) | 10.2.0.0/16 | 10.2.0.0/15 |
| резерв (/16) | (8*(N-1) + 3) | 10.3.0.0/16 |  |
| services (/16) | (8*(N-1) + [4 .. 7]) | 10.4.0.0/16, 10.5.0.0/16, 10.6.0.0/16, 10.7.0.0/16 | 10.4.0.0/14 |

```
Подход к распределение IP адресов для DC1
10.0.1.0/32 – Spine 1 Lo0
...
10.0.N.0/32 – Spine N Lo0

10.1.0.1/32 – Leaf 1 Lo1
...
10.1.0.N/32 – Leaf N Lo1

10.2.2.4/31 – p2p Spine 2 <-> Leaf 3
10.2.N.2*(M-1)/31 – p2p Spine N <-> Leaf M, где M= [1 .. 128]
```
2.3. Итоговый план адресации:
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
!
interface Loopback1
   ip address 10.1.1.0/32
!
interface Management1
!
ip routing
!
end
Spine-1#
```
</details>

<details>
<summary> Spine-2 </summary>
  
```
Spine-2#sh run
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
!
interface Loopback1
   ip address 10.1.2.0/32
!
interface Management1
!
ip routing
!
end
Spine-2#

```
</details>

<details>
<summary> Leaf-1 </summary>
  
```
Leaf-1#sh run
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
end
Leaf-2#
```
</details>

<details>
<summary> Leaf-3 </summary>
  
```
Leaf-3#sh run
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
end
Leaf-3#

```
</details>

<details>
<summary> Servers 1-4 </summary>
  
```
Server-1> show ip
NAME        : Server-1[1]
IP/MASK     : 10.4.1.2/24
GATEWAY     : 10.4.1.1
DNS         :
MAC         : 00:50:79:66:68:06
LPORT       : 20000
RHOST:PORT  : 127.0.0.1:30000
MTU         : 1500
Server-1>

Server-2> show ip
NAME        : Server-2[1]
IP/MASK     : 10.4.2.2/24
GATEWAY     : 10.4.2.1
DNS         :
MAC         : 00:50:79:66:68:07
LPORT       : 20000
RHOST:PORT  : 127.0.0.1:30000
MTU         : 1500
Server-2>

Server-3> show ip
NAME        : Server-3[1]
IP/MASK     : 10.4.3.2/24
GATEWAY     : 10.4.3.1
DNS         :
MAC         : 00:50:79:66:68:08
LPORT       : 20000
RHOST:PORT  : 127.0.0.1:30000
MTU         : 1500
Server-3>

Server-4> show ip
NAME        : Server-4[1]
IP/MASK     : 10.4.4.2/24
GATEWAY     : 10.4.4.1
DNS         :
MAC         : 00:50:79:66:68:09
LPORT       : 20000
RHOST:PORT  : 127.0.0.1:30000
MTU         : 1500
Server-4>

```
</details>













