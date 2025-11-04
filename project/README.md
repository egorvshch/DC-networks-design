## Проектная работа
Проектирование резервного ЦОД с использованием технологий VxLAN EVPN
***
### Цель:
- Организовать резервирование сервисов, находящихся в продуктиве (vrf BLUE), между двумя ЦОД на уровне L2 и L3.
- Организовать взаимодействие сервисов между двумя ЦОД на уровне L3 в пределах своих vrf напрямую, без внешнего маршрутизатора или МСЭ 
#### Задачи:
1. Объединить два ЦОД находящихся каждый в своем домене EVPN VXLAN через L3.
2. Организовать L2 и L3 связность между EVPN доменами разных ЦОД 
3. Обеспечить отказоустойчивость и масштабируемость соединения между ЦОД
4. Обеспечить отказоустойчивость на уровне доступа 
***
## Результаты выполнения:
---
В качестве решения организации интерконнекта между ЦОД используется Multi-Domain EVPN VXLAN (Anycast Gateway), с резервированием пограничных шлюзов (Border GW).

Отказоустойчивость на уровне доступа реализуется с помощью пары MLAG.
Underlay и Overlay на eBGP

<details>
<summary> Распределение адресных пространств и ASN </summary>

**DC1**:
```
ASN: AS65100 - AS65199
Подсеть на ЦОД: 10.0.0.0/13
Loopback0 (Underlay) Spine: 10.0.х.0/32, х - номер Spine
Loopback1 (Underlay) Spine: 10.1.х.0/32, х - номер Spine
Loopback0 (Underlay) Leaf: 10.0.0.x/32, х - номер Leaf
Loopback1 (Underlay) Leaf: 10.1.0.х/32, х - номер Leaf
P2P: 10.2.x.y/16 - х - номер Spine, y - номер по порядку 
Сервисы:
- vrf BLUE (VNI 11001), VLAN 10 (VNI 10010) IP: 10.4.10.0/24, Anycast GW: 10.4.10.254;
- vrf RED (VNI 11002), VLAN 20 (VNI 10020) IP: 10.5.20.0/24, Anycast GW: 10.5.20.254
```
**DC2**:
```
ASN: AS65200 - AS65299
Подсеть на ЦОД: 10.8.0.0/13
Loopback0 (Underlay) Spine: 10.8.х.0/32, х - номер Spine
Loopback1 (Underlay) Spine: 10.9.х.0/32, х - номер Spine
Loopback0 (Underlay) Leaf: 10.8.0.x/32, х - номер Leaf
Loopback1 (Underlay) Leaf: 10.9.0.х/32, х - номер Leaf
P2P: 10.10.x.y/16, х - номер Spine, y - номер по порядку 
Сервисы:
vrf BLUE (VNI 11001):
- VLAN 10 (VNI 10010) IP: 10.4.10.0/24, Anycast GW: 10.4.10.254
- VLAN 14 (VNI 10014) IP: 10.14.14.0/24, Anycast GW: 10.14.14.254
- vrf RED (VNI 11002): VLAN 20 (VNI 10020) IP: 10.12.20.0/24, Anycast GW: 10.12.20.254
- vrf GREEN (VNI 11003): VLAN 30 (VNI 10030) IP: 10.13.30.0/24, Anycast GW: 10.13.30.254
```
**DCI:**
```
ASN: AS65999
Loopback0 (Underlay): 10.91.91.x/32, х - номер DCI маршрутизатора
P2P: 10.2.9x.y/31, х - номер DCI маршрутизатора, y - номер по порядку 
```
**DC-Border-R:**
```
ASN: AS65301
Loopback0 (Underlay): 10.199.0.x/32, х - номер маршрутизатора
P2P между DC-Border-R: 10.195.0.x/30, х - номер по порядку 
Стыковочные сервисные сети внутри DC:
vrf BLUE: vlan 104, 10.19х.104.y/30 - х=8 - Border-GW1, х=9 - - Border-GW2, y - номер по порядку 
vrf RED: vlan 105, 10.19х.105.y/30 - х=8 - Border-GW1, х=9 - - Border-GW2, y - номер по порядку 
vrf GREEN: vlan 106, 10.19х.106.y/30 - х=8 - Border-GW1, х=9 - - Border-GW2, y - номер по порядку 
Стыковочные сервисные сети между DC:
vrf BLUE: vlan 114, 10.19х.104.y/30 - х=8 - Border-GW1, х=9 - - Border-GW2, y - номер по порядку 
vrf RED: vlan 115, 10.19х.105.y/30 - х=8 - Border-GW1, х=9 - - Border-GW2, y - номер по порядку 
vrf GREEN: vlan 116, 10.19х.106.y/30 - х=8 - Border-GW1, х=9 - - Border-GW2, y - номер по порядку 
```
</details>

### 1. Схема сети:
   
![](https://github.com/egorvshch/DC-networks-design/blob/main/project/images/schema_dc_network-.jpg)
---

### 2. IP-адресация оборудования:

<details>
<summary> Таблица IP-адресация оборудования DC1 </summary>
   
| Device | Interface	| IP Address | Subnet Mask | description |
|:------ |:-----------|:----------:|:-------------:|:-------------:|
| Spine-1	| Lo0	| 10.0.1.0	| /32 | Underlay |
| | Lo1	| 10.1.1.0	| /32 | Overlay |
| | Eth1	| 10.2.1.0	| /31 | to Leaf-1-1 |
| | Eth2	| 10.2.1.2	| /31 | to Leaf-1-2 |
| | Eth3	| 10.2.1.4	| /31 | to Leaf-2-1 |
| | Eth4	| 10.2.1.6	| /31 | to Leaf-2-2 |
| | Eth5	| 10.2.1.8	| /31 | to Border-GW1 |
| | Eth6	| 10.2.1.10	| /31 | to Border-GW2 |
| Spine-2	| Lo0	| 10.0.2.0	| /32 | Underlay |
| | Lo1	| 10.1.2.0	| /32 | Overlay |
| | Eth1	| 10.2.2.0	| /31 | to Leaf-1-1 |
| | Eth2	| 10.2.2.2	| /31 | to Leaf-1-2 |
| | Eth3	| 10.2.2.4	| /31 | to Leaf-2-1 |
| | Eth4	| 10.2.2.6	| /31 | to Leaf-2-2 |
| | Eth5	| 10.2.2.8	| /31 | to Border-GW1 |
| | Eth6	| 10.2.2.10	| /31 | to Border-GW2 |
| Leaf-1-1	| Lo0	| 10.0.0.1	| /32 | Underlay |
| | Lo1	| 10.1.0.1	| /32 | Overlay |
| | Eth1	| 10.2.1.1	| /31 | to Spine-1 |
| | Eth2	| 10.2.2.1	|/31 | to Spine-2 |
| | Eth3	| 10.199.250.1/30	| vlan 4090 | mlag Po999 MLAG Peer-address to Leaf-1-2 |
| | | 10.199.251.1/30	| vlan 4091 | mlag Po999 for_iBGP to Leaf-1-2 |
| | Eth4 | 10.199.250.1/30	| vlan 4090 | mlag Po999 MLAG Peer-address to Leaf-1-2 |
| | | 10.199.251.1/30	| vlan 4091 | mlag Po999 for_iBGP to Leaf-1-2 |
| | Eth5	| 	| vlan 10 | to Host-1 |
| | Eth6 | 	| vlan 20 | to Host-2 |
| Leaf-1-2	| Lo0	| 10.0.0.2	| /32 |
| | Lo1	| 10.1.0.2	| /32 |
| | Eth1	| 10.2.1.3	| /31 | to Spine-1 |
| | Eth2	| 10.2.2.3	| /31 | to Spine-2 |
| | Eth3	| 10.199.250.2/30	| vlan 4090 | mlag Po999 MLAG Peer-address to Leaf-1-1 |
| | | 10.199.251.2/30	| vlan 4091 | mlag Po999 for_iBGP to Leaf-1-1 |
| | Eth4 | 10.199.250.2/30	| vlan 4090 | mlag Po999 MLAG Peer-address to Leaf-1-1 |
| | | 10.199.251.2/30	| vlan 4091 | mlag Po999 for_iBGP to Leaf-1-1 |
| | Eth5	| 	| vlan 10 | Po10 to Host-1 |
| | Eth6 | 	| vlan 20 | Po20 to Host-2 |
| Leaf-2-1	| Lo0	| 10.0.0.3	| /32 | Underlay |
| | Lo1	| 10.1.0.3	| /32 | Overlay |
| | Eth1	| 10.2.1.5	| /31 | to Spine-1 |
| | Eth2	| 10.2.2.5	|/31 | to Spine-2 |
| | Eth3	| 10.199.250.5/30	| vlan 4090 | mlag Po999 MLAG Peer-address to Leaf-2-2 |
| | | 10.199.251.5/30	| vlan 4091 | mlag Po999 for_iBGP to Leaf-2-2 |
| | Eth4 | 10.199.250.5/30	| vlan 4090 | mlag Po999 MLAG Peer-address to Leaf-2-2 |
| | | 10.199.251.5/30	| vlan 4091 | mlag Po999 for_iBGP to Leaf-2-2 |
| | Eth5	| 	| vlan 10 | to Host-3 |
| | Eth6 | 	| vlan 20 | to Host-4 |
| Leaf-2-2	| Lo0	| 10.0.0.4	| /32 | Underlay |
| | Lo1	| 10.1.0.4	| /32 | Overlay |
| | Eth1	| 10.2.1.7	| /31 | to Spine-1 |
| | Eth2	| 10.2.2.7	|/31 | to Spine-2 |
| | Eth3	| 10.199.250.6/30	| vlan 4090 | mlag Po999 MLAG Peer-address to Leaf-2-1 |
| | | 10.199.251.6/30	| vlan 4091 | mlag Po999 for_iBGP to Leaf-2-1 |
| | Eth4 | 10.199.250.6/30	| vlan 4090 | mlag Po999 MLAG Peer-address to Leaf-2-1 |
| | | 10.199.251.6/30	| vlan 4091 | mlag Po999 for_iBGP to Leaf-2-1 |
| | Eth5	| 	| vlan 10 | to Host-3 |
| | Eth6 | 	| vlan 20 | to Host-4 |
| Border-GW1	| Lo0	| 10.0.0.5	| /32 | Underlay |
| | Lo1	| 10.1.0.5	| /32 | Overlay |
| | Eth1	| 10.2.1.9	| /31 | to Spine-1 |
| | Eth2	| 10.2.2.9	| /31 | to Spine-2 |
| | Eth3	|  |  | to DC1-Border-R1 |
| | Eth3.104 | 10.198.104.1/30	| vlan 104 | vrf BLUE |
| | Eth3.105 | 10.198.105.1/30	| vlan 105 | vrf RED |
| | Eth3.106 | 10.198.106.1/30	| vlan 105 | vrf GREEN |
| | Eth4	| 10.2.91.1/31 |  | to DCI-Router-1 |
| | Eth5	| 10.2.92.1/31 |  | to DCI-Router-2 |
| | Eth6	|  |  | to DC2-Border-R1 |
| | Eth6.114 | 10.198.104.9/30	| vlan 114 | vrf BLUE |
| | Eth6.115 | 10.198.105.9/30	| vlan 115 | vrf RED |
| | Eth6.116 | 10.198.106.9/30	| vlan 115 | vrf GREEN |
| Border-GW2	| Lo0	| 10.0.0.6	| /32 | Underlay |
| | Lo1	| 10.1.0.5	| /32 | Overlay |
| | Eth1	| 10.2.1.11	| /31 | to Spine-1 |
| | Eth2	| 10.2.2.11	| /31 | to Spine-2 |
| | Eth3	|  |  | to DC1-Border-R1 |
| | Eth3.104 | 10.199.104.1/30	| vlan 104 | vrf BLUE |
| | Eth3.105 | 10.199.105.1/30	| vlan 105 | vrf RED |
| | Eth3.106 | 10.199.106.1/30	| vlan 105 | vrf GREEN |
| | Eth4	| 10.2.91.3/31 |  | to DCI-Router-1 |
| | Eth5	| 10.2.92.3/31 |  | to DCI-Router-2 |
| | Eth6	|  |  | to DC2-Border-R1 |
| | Eth6.114 | 10.198.104.9/30	| vlan 114 | vrf BLUE |
| | Eth6.115 | 10.198.105.9/30	| vlan 115 | vrf RED |
| | Eth6.116 | 10.198.106.9/30	| vlan 115 | vrf GREEN |
| Host-1	| eth0	| 10.4.10.11	| /24 | Vlan 10 / vrf BLUE |
| Host-2	| eth0	| 10.5.20.12	| /24 | Vlan 20 / vrf RED |
| Host-3	| eth0	| 10.4.10.13	| /24 | Vlan 10 / vrf BLUE |
| Host-4	| eth0	| 10.5.20.14	| /24 | Vlan 20 / vrf RED |

</details>

<details>
<summary> Таблица IP-адресация оборудования DC2 </summary>
   
| Device | Interface	| IP Address | Subnet Mask | description |
|:------ |:-----------|:----------:|:-------------:|:-------------:|
| Spine-1	| Lo0	| 10.8.1.0	| /32 | Underlay |
| | Lo1	| 10.8.1.0	| /32 | Overlay |
| | Eth1	| 10.10.1.0	| /31 | to Leaf-1-1 |
| | Eth2	| 10.10.1.2	| /31 | to Leaf-1-2 |
| | Eth3	| 10.10.1.4	| /31 | to Leaf-2-1 |
| | Eth4	| 10.10.1.6	| /31 | to Leaf-2-2 |
| | Eth5	| 10.10.1.8	| /31 | to Border-GW1 |
| | Eth6	| 10.10.1.10	| /31 | to Border-GW2 |
| Spine-2	| Lo0	| 10.8.2.0	| /32 | Underlay |
| | Lo1	| 10.9.2.0	| /32 | Overlay |
| | Eth1	| 10.10.2.0	| /31 | to Leaf-1-1 |
| | Eth2	| 10.10.2.2	| /31 | to Leaf-1-2 |
| | Eth3	| 10.10.2.4	| /31 | to Leaf-2-1 |
| | Eth4	| 10.10.2.6	| /31 | to Leaf-2-2 |
| | Eth5	| 10.10.2.8	| /31 | to Border-GW1 |
| | Eth6	| 10.10.2.10	| /31 | to Border-GW2 |
| Leaf-1-1	| Lo0	| 10.8.0.1	| /32 | Underlay |
| | Lo1	| 10.9.0.1	| /32 | Overlay |
| | Eth1	| 10.10.1.1	| /31 | to Spine-1 |
| | Eth2	| 10.10.2.1	|/31 | to Spine-2 |
| | Eth3	| 10.199.252.1/30	| vlan 4090 | mlag Po999 MLAG Peer-address to Leaf-1-2 |
| | | 10.199.253.1/30	| vlan 4091 | mlag Po999 for_iBGP to Leaf-1-2 |
| | Eth4 | 10.199.252.1/30	| vlan 4090 | mlag Po999 MLAG Peer-address to Leaf-1-2 |
| | | 10.199.253.1/30	| vlan 4091 | mlag Po999 for_iBGP to Leaf-1-2 |
| | Eth5	| 	| vlan 10 | to Host-1 |
| | Eth6 | 	| vlan 30 | to Host-2 |
| Leaf-1-2	| Lo0	| 10.8.0.2	| /32 |
| | Lo1	| 10.9.0.2	| /32 |
| | Eth1	| 10.10.1.3	| /31 | to Spine-1 |
| | Eth2	| 10.10.2.3	| /31 | to Spine-2 |
| | Eth3	| 10.199.252.2/30	| vlan 4090 | mlag Po999 MLAG Peer-address to Leaf-1-1 |
| | | 10.199.253.2/30	| vlan 4091 | mlag Po999 for_iBGP to Leaf-1-1 |
| | Eth4 | 10.199.252.2/30	| vlan 4090 | mlag Po999 MLAG Peer-address to Leaf-1-1 |
| | | 10.199.253.2/30	| vlan 4091 | mlag Po999 for_iBGP to Leaf-1-1 |
| | Eth5	| 	| vlan 10 | Po10 to Host-1 |
| | Eth6 | 	| vlan 30 | Po20 to Host-2 |
| Leaf-2-1	| Lo0	| 10.8.0.3	| /32 | Underlay |
| | Lo1	| 10.9.0.3	| /32 | Overlay |
| | Eth1	| 10.10.1.5	| /31 | to Spine-1 |
| | Eth2	| 10.10.2.5	|/31 | to Spine-2 |
| | Eth3	| 10.199.252.5/30	| vlan 4090 | mlag Po999 MLAG Peer-address to Leaf-2-2 |
| | | 10.199.253.5/30	| vlan 4091 | mlag Po999 for_iBGP to Leaf-2-2 |
| | Eth4 | 10.199.252.5/30	| vlan 4090 | mlag Po999 MLAG Peer-address to Leaf-2-2 |
| | | 10.199.253.5/30	| vlan 4091 | mlag Po999 for_iBGP to Leaf-2-2 |
| | Eth5	| 	| vlan 30 | to Host-3 |
| | Eth6 | 	| vlan 20 | to Host-4 |
| Leaf-2-2	| Lo0	| 10.8.0.4	| /32 | Underlay |
| | Lo1	| 10.9.0.4	| /32 | Overlay |
| | Eth1	| 10.10.1.7	| /31 | to Spine-1 |
| | Eth2	| 10.10.2.7	|/31 | to Spine-2 |
| | Eth3	| 10.199.252.6/30	| vlan 4090 | mlag Po999 MLAG Peer-address to Leaf-2-1 |
| | | 10.199.253.6/30	| vlan 4091 | mlag Po999 for_iBGP to Leaf-2-1 |
| | Eth4 | 10.199.253.6/30	| vlan 4090 | mlag Po999 MLAG Peer-address to Leaf-2-1 |
| | | 10.199.253.6/30	| vlan 4091 | mlag Po999 for_iBGP to Leaf-2-1 |
| | Eth5	| 	| vlan 30 | to Host-3 |
| | Eth6 | 	| vlan 20 | to Host-4 |
| Border-GW1	| Lo0	| 10.8.0.5	| /32 | Underlay |
| | Lo1	| 10.9.0.5	| /32 | Overlay |
| | Eth1	| 10.10.1.9	| /31 | to Spine-1 |
| | Eth2	| 10.10.2.9	| /31 | to Spine-2 |
| | Eth3	|  |  | to DC2-Border-R1 |
| | Eth3.104 | 10.198.104.5/30	| vlan 104 | vrf BLUE |
| | Eth3.105 | 10.198.105.5/30	| vlan 105 | vrf RED |
| | Eth3.106 | 10.198.106.5/30	| vlan 105 | vrf GREEN |
| | Eth4	| 10.2.91.5/31 |  | to DCI-Router-1 |
| | Eth5	| 10.2.92.5/31 |  | to DCI-Router-2 |
| | Eth6	|  |  | to DC1-Border-R1 |
| | Eth6.114 | 10.198.104.13/30	| vlan 114 | vrf BLUE |
| | Eth6.115 | 10.198.105.13/30	| vlan 115 | vrf RED |
| | Eth6.116 | 10.198.106.13/30	| vlan 115 | vrf GREEN |
| Border-GW2	| Lo0	| 10.8.0.6	| /32 | Underlay |
| | Lo1	| 10.9.0.5	| /32 | Overlay |
| | Eth1	| 10.10.1.11	| /31 | to Spine-1 |
| | Eth2	| 10.10.2.11	| /31 | to Spine-2 |
| | Eth3	|  |  | to DC2-Border-R1 |
| | Eth3.104 | 10.199.104.5/30	| vlan 104 | vrf BLUE |
| | Eth3.105 | 10.199.105.5/30	| vlan 105 | vrf RED |
| | Eth3.106 | 10.199.106.5/30	| vlan 105 | vrf GREEN |
| | Eth4	| 10.2.91.7/31 |  | to DCI-Router-1 |
| | Eth5	| 10.2.92.7/31 |  | to DCI-Router-2 |
| | Eth6	|  |  | to DC1-Border-R1 |
| | Eth6.114 | 10.198.104.13/30	| vlan 114 | vrf BLUE |
| | Eth6.115 | 10.198.105.13/30	| vlan 115 | vrf RED |
| | Eth6.116 | 10.198.106.13/30	| vlan 115 | vrf GREEN |
| Host-1	| eth0	| 10.4.10.21	| /24 | Vlan 10 / vrf BLUE |
| 	| 	| 10.14.14.254	| /24 | Vlan 14 / vrf BLUE |
| Host-2	| eth0	| 10.13.30.22	| /24 | Vlan 30 / vrf GREEN |
| Host-3	| eth0	| 10.13.30.23	| /24 | Vlan 30 / vrf GREEN |
| Host-4	| eth0	| 10.12.20.24	| /24 | Vlan 20 / vrf RED |

</details>

<details>
<summary> Таблица IP-адресация оборудования DCI и Border Routers </summary>
   
| Device | Interface	| IP Address | Subnet Mask | description |
|:------ |:-----------|:----------:|:-------------:|:-------------:|
| DCI-Router-1	| Lo0	| 10.91.91.1	| /32 | Underlay |
| | Eth1	| 10.2.91.0	| /31 | to DC1-Border-GW1|
| | Eth2	| 10.2.91.2	| /31 | to DC1-Border-GW2 |
| | Eth3	| 10.2.91.4	| /31 | to DC2-Border-GW1|
| | Eth4	| 10.2.91.6	| /31 | to DC2-Border-GW2 |
| DCI-Router-2	| Lo0	| 10.91.91.2	| /32 | Underlay |
| | Eth1	| 10.2.92.0	| /31 | to DC1-Border-GW1|
| | Eth2	| 10.2.92.2	| /31 | to DC1-Border-GW2 |
| | Eth3	| 10.2.92.4	| /31 | to DC2-Border-GW1|
| | Eth4	| 10.2.92.6	| /31 | to DC2-Border-GW2 |
| DC1-Border-R1	| Lo0	| 10.199.0.1	| /32 | Underlay |
| | Lo11	| 1.1.1.1	| /24 | Internet |
| | Eth1	| |  | to DC1-Border-GW1 |
| | Eth1.104 | 10.198.104.2	| /30 | to vrf_BLUE_for_DC1-Border-GW1 |
| | Eth1.105 | 10.198.105.2	| /30 | to vrf_RED_for_DC1-Border-GW1|
| | Eth1.106 | 10.198.106.2	| /30 | to vrf_GREEN_for_DC1-Border-GW1 |
| | Eth2	| |  | to DC1-Border-GW2 |
| | Eth2.104 | 10.199.104.2	| /30 | to vrf_BLUE_for_DC1-Border-GW2 |
| | Eth2.105 | 10.199.105.2	| /30 | to vrf_RED_for_DC1-Border-GW2 |
| | Eth2.106 | 10.199.106.2	| /30 | to vrf_GREEN_for_DC1-Border-GW2 |
| | Eth3	| |  | to DC2-Border-GW1 |
| | Eth3.114 | 10.198.104.14	| /30 | to vrf_BLUE_for_DC2-Border-GW1 |
| | Eth3.115 | 10.198.105.14	| /30 | to vrf_RED_for_DC2-Border-GW1|
| | Eth3.116 | 10.198.106.14	| /30 | to vrf_GREEN_for_DC2-Border-GW1 |
| | Eth4	| |  | to DC2-Border-GW2 |
| | Eth4.114 | 10.199.104.14	| /30 | to vrf_BLUE_for_DC2-Border-GW2 |
| | Eth4.115 | 10.199.105.14	| /30 | to vrf_RED_for_DC2-Border-GW2 |
| | Eth4.116 | 10.199.106.14	| /30 | to vrf_GREEN_for_DC2-Border-GW2 |
| | Eth5	| 10.195.0.1 | /30 | to DC2-Border-R1 |
| DC2-Border-R1	| Lo0	| 10.199.0.2	| /32 | Underlay |
| | Lo11	| 2.2.2.2	| /32 | Internet |
| | Eth1	| |  | to DC2-Border-GW1 |
| | Eth1.104 | 10.198.104.6	| /30 | to vrf_BLUE_for_DC2-Border-GW1 |
| | Eth1.105 | 10.198.105.6	| /30 | to vrf_RED_for_DC2-Border-GW1|
| | Eth1.106 | 10.198.106.6	| /30 | to vrf_GREEN_for_DC2-Border-GW1 |
| | Eth2	| |  | to DC2-Border-GW2 |
| | Eth2.104 | 10.199.104.6	| /30 | to vrf_BLUE_for_DC2-Border-GW2 |
| | Eth2.105 | 10.199.105.6	| /30 | to vrf_RED_for_DC2-Border-GW2 |
| | Eth2.106 | 10.199.106.6	| /30 | to vrf_GREEN_for_DC2-Border-GW2 |
| | Eth3	| |  | to DC1-Border-GW1 |
| | Eth3.114 | 10.198.104.10	| /30 | to vrf_BLUE_for_DC1-Border-GW1 |
| | Eth3.115 | 10.198.105.10	| /30 | to vrf_RED_for_DC1-Border-GW1|
| | Eth3.116 | 10.198.106.10	| /30 | to vrf_GREEN_for_DC1-Border-GW1 |
| | Eth4	| |  | to DC1-Border-GW2 |
| | Eth4.114 | 10.199.104.10	| /30 | to vrf_BLUE_for_DC1-Border-GW2 |
| | Eth4.115 | 10.199.105.10	| /30 | to vrf_RED_for_DC1-Border-GW2 |
| | Eth4.116 | 10.199.106.10	| /30 | to vrf_GREEN_for_DC1-Border-GW2 |
| | Eth5	| 10.195.0.2 | /30 | to DC1-Border-R1 |
</details>

---
### 3. Настройка оборудрования DC1:
<details>
<summary> 3.1 Настройка DC1-Spine-1 </summary>
*
<details>
<summary> Базовые настройки и настройки интерфейсов на DC1-Spine-1 </summary>

```  
service routing protocols model multi-agent
!
hostname DC1-Spine-1
!
spanning-tree mode mstp
!
interface Ethernet1
   description to DC1-Leaf-1-1
   no switchport
   ip address 10.2.1.0/31
!
interface Ethernet2
   description to DC1-Leaf-1-2
   no switchport
   ip address 10.2.1.2/31
!
interface Ethernet3
   description to DC1-Leaf-2-1
   no switchport
   ip address 10.2.1.4/31
!
interface Ethernet4
   description to DC1-Leaf-2-2
   no switchport
   ip address 10.2.1.6/31
!
interface Ethernet5
   description to DC1-Border-GW1
   no switchport
   ip address 10.2.1.8/31
!
interface Ethernet6
   description to DC1-Border-GW2
   no switchport
   ip address 10.2.1.10/31
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
<summary> Настройки маршрутизации BGP, Underlay и Overlay на DC1-Spine-1 </summary>

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
peer-filter OVERLAY
   10 match as-range 65101-65199 result accept
!
peer-filter UNDERLAY
   10 match as-range 65101-65199 result accept
!
router bgp 65100
   bgp asn notation asdot
   router-id 10.0.1.0
   timers bgp 3 9
   maximum-paths 4 ecmp 4
   bgp listen range 10.0.0.0/24 peer-group OVERLAY peer-filter OVERLAY
   bgp listen range 10.2.1.0/27 peer-group UNDERLAY peer-filter UNDERLAY
   neighbor OVERLAY peer group
   neighbor OVERLAY next-hop-unchanged
   neighbor OVERLAY update-source Loopback0
   neighbor OVERLAY ebgp-multihop 3
   neighbor OVERLAY send-community extended
   neighbor UNDERLAY peer group
   neighbor UNDERLAY password 7 4vMwMsSftvmv7pRrLIgVxA==
   neighbor UNDERLAY send-community
   redistribute connected route-map RM_REDISTRIBUTE_CONNECTED
   !
   address-family evpn
      neighbor OVERLAY activate
   !
   address-family ipv4
      no neighbor OVERLAY activate
!
```
</details>

[Полная конфигурация DC1-Spine-1](https://github.com/egorvshch/DC-networks-design/blob/main/project/configs/DC1-Spine-1)

</details>

<details>
<summary> 3.2 Настройка DC1-Spine-2 </summary>
*
<details>
<summary> Базовые настройки и настройки интерфейсов на DC1-Spine-2 </summary>

```  
service routing protocols model multi-agent
!
hostname DC1-Spine-2
!
!
interface Ethernet1
   description to DC1-Leaf-1-1
   no switchport
   ip address 10.2.2.0/31
!
interface Ethernet2
   description to DC1-Leaf-1-2
   no switchport
   ip address 10.2.2.2/31
!
interface Ethernet3
   description to DC1-Leaf-2-1
   no switchport
   ip address 10.2.2.4/31
!
interface Ethernet4
   description to DC1-Leaf-2-2
   no switchport
   ip address 10.2.2.6/31
!
interface Ethernet5
   description to DC1-Border-GW1
   no switchport
   ip address 10.2.2.8/31
!
interface Ethernet6
   description to DC1-Border-GW1
   no switchport
   ip address 10.2.2.10/31
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
<summary> Настройки маршрутизации BGP, Underlay и Overlay на DC1-Spine-2 </summary>

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
peer-filter OVERLAY
   10 match as-range 65101-65199 result accept
!
peer-filter UNDERLAY
   10 match as-range 65101-65199 result accept
!
router bgp 65100
   bgp asn notation asdot
   router-id 10.0.2.0
   timers bgp 3 9
   maximum-paths 4 ecmp 4
   bgp listen range 10.0.0.0/24 peer-group OVERLAY peer-filter OVERLAY
   bgp listen range 10.2.2.0/24 peer-group UNDERLAY peer-filter UNDERLAY
   neighbor OVERLAY peer group
   neighbor OVERLAY next-hop-unchanged
   neighbor OVERLAY update-source Loopback0
   neighbor OVERLAY ebgp-multihop 3
   neighbor OVERLAY send-community extended
   neighbor UNDERLAY peer group
   neighbor UNDERLAY password 7 4vMwMsSftvmv7pRrLIgVxA==
   neighbor UNDERLAY send-community
   redistribute connected route-map RM_REDISTRIBUTE_CONNECTED
   !
   address-family evpn
      neighbor OVERLAY activate
   !
   address-family ipv4
      no neighbor OVERLAY activate
!
```
</details>

[Полная конфигурация DC1-Spine-2](https://github.com/egorvshch/DC-networks-design/blob/main/project/configs/DC1-Spine-2)
</details>

<details>
<summary> 3.3 Настройка DC1-Leaf-1-1 </summary>
-

<details>
<summary> Базовые настройки и настройки интерфейсов на DC1-Leaf-1-1 </summary>

```  
service routing protocols model multi-agent
!
hostname DC1-Leaf-1-1
!
spanning-tree mode mstp
no spanning-tree vlan-id 4090-4091
!
vlan 10
   name service-10
!
vlan 20
   name service-20
!
vlan 4090
   name mlag-peer
   trunk group mlag-peer
!
vlan 4091
   name mlag-iBGP
   trunk group mlag-peer
!
vrf instance BLUE
!
vrf instance RED
!
interface Port-Channel10
   description to Host-1
   switchport trunk allowed vlan 10
   switchport mode trunk
   mlag 10
!
interface Port-Channel20
   description to Host-2
   switchport trunk allowed vlan 20
   switchport mode trunk
   mlag 20
!
interface Port-Channel999
   description mlag_peer-link_to_Leaf-1-2
   switchport mode trunk
   switchport trunk group mlag-peer
!
interface Ethernet1
   description to DC1-Spine-1
   no switchport
   ip address 10.2.1.1/31
!
interface Ethernet2
   description to DC1-Spine-2
   no switchport
   ip address 10.2.2.1/31
!
interface Ethernet3
   description mlag_peer-link_to_Leaf-1-2_Eth4
   switchport trunk allowed vlan 10
   switchport mode trunk
   channel-group 999 mode active
!
interface Ethernet4
   description mlag_peer-link_to_Leaf-1-2_Eth5
   switchport trunk allowed vlan 20
   switchport mode trunk
   channel-group 999 mode active
!
interface Ethernet5
   description Host-1_eth0/0
   switchport trunk allowed vlan 10
   switchport mode trunk
   channel-group 10 mode active
!
interface Ethernet6
   description Host-2_eth0/0
   switchport trunk allowed vlan 20
   switchport mode trunk
   channel-group 20 mode active
!
!
interface Loopback0
   ip address 10.0.0.1/32
!
interface Loopback1
   ip address 10.1.0.1/32
!
!
interface Vlan10
   vrf BLUE
   ip address virtual 10.4.10.254/24
!
interface Vlan20
   vrf RED
   ip address virtual 10.5.20.254/24
!
interface Vlan4090
   description MLAG Peer-address
   ip address 10.199.250.1/30
!
interface Vlan4091
   description for_iBGP
   ip address 10.199.251.1/30
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
   vxlan vrf BLUE vni 11001
   vxlan vrf RED vni 11002
   vxlan learn-restrict any
!
ip virtual-router mac-address 00:00:00:00:00:01
!
```
</details>

<details>
<summary> Настройки маршрутизации BGP, Underlay и Overlay на DC1-Leaf-1-1 </summary>

```  
ip routing
ip routing vrf BLUE
ip routing vrf RED
!
ip prefix-list DIRECT_CONNECTED
   seq 10 permit 10.0.0.1/32
   seq 20 permit 10.1.0.1/32
!
mlag configuration
   domain-id mlag-999
   local-interface Vlan4090
   peer-address 10.199.250.2
   peer-link Port-Channel999
!
route-map RM_REDISTRIBUTE_CONNECTED permit 10
   match ip address prefix-list DIRECT_CONNECTED
!
router bgp 65101
   router-id 10.0.0.1
   timers bgp 3 9
   maximum-paths 4 ecmp 4
   neighbor OVERLAY peer group
   neighbor OVERLAY remote-as 65100
   neighbor OVERLAY update-source Loopback0
   neighbor OVERLAY ebgp-multihop 3
   neighbor OVERLAY send-community extended
   neighbor UNDERLAY peer group
   neighbor UNDERLAY remote-as 65100
   neighbor UNDERLAY password 7 4vMwMsSftvmv7pRrLIgVxA==
   neighbor UNDERLAY send-community
   neighbor 10.0.1.0 peer group OVERLAY
   neighbor 10.0.2.0 peer group OVERLAY
   neighbor 10.2.1.0 peer group UNDERLAY
   neighbor 10.2.2.0 peer group UNDERLAY
   neighbor 10.199.251.2 remote-as 65101
   neighbor 10.199.251.2 next-hop-self
   redistribute connected route-map RM_REDISTRIBUTE_CONNECTED
   !
   vlan 10
      rd 65101:10010
      route-target both 10:10010
      redistribute learned
   !
   vlan 20
      rd 65101:10020
      route-target both 20:10020
      redistribute learned
   !
   address-family evpn
      neighbor OVERLAY activate
   !
   address-family ipv4
      no neighbor OVERLAY activate
      neighbor 10.199.251.2 activate
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
end
```
</details>

[Полная конфигурация DC1-Leaf-1-1](https://github.com/egorvshch/DC-networks-design/blob/main/project/configs/DC1-Leaf-1-1)
</details>

<details>
<summary> 3.4 Настройка DC1-Leaf-1-2 </summary>
-

<details>
<summary> Базовые настройки и настройки интерфейсов на DC1-Leaf-1-2 </summary>

```  
!
service routing protocols model multi-agent
!
hostname DC1-Leaf-1-2
!
spanning-tree mode mstp
no spanning-tree vlan-id 4090-4091
!
vlan 10
   name service-10
!
vlan 20
   name service-20
!
vlan 4090
   name mlag-peer
   trunk group mlag-peer
!
vlan 4091
   name mlag-iBGP
   trunk group mlag-peer
!
vrf instance BLUE
!
vrf instance RED
!
interface Port-Channel10
   description to Host-1
   switchport trunk allowed vlan 10
   switchport mode trunk
   mlag 10
!
interface Port-Channel20
   description to Host-2
   switchport trunk allowed vlan 20
   switchport mode trunk
   mlag 20
!
interface Port-Channel999
   description mlag_peer-link_to_Leaf-1-2
   switchport mode trunk
   switchport trunk group mlag-peer
!
interface Ethernet1
   description to DC1-Spine-1
   no switchport
   ip address 10.2.1.3/31
!
interface Ethernet2
   description to DC1-Spine-2
   no switchport
   ip address 10.2.2.3/31
!
interface Ethernet3
   description mlag_peer-link_to_Leaf-1-2_Eth4
   channel-group 999 mode active
!
interface Ethernet4
   description mlag_peer-link_to_Leaf-1-2_Eth5
   channel-group 999 mode active
!
interface Ethernet5
   description Host-1_eth0/0
   switchport trunk allowed vlan 10
   switchport mode trunk
   channel-group 10 mode active
!
interface Ethernet6
   description Host-2_eth0/0
   switchport trunk allowed vlan 20
   switchport mode trunk
   channel-group 20 mode active
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
interface Vlan10
   vrf BLUE
   ip address virtual 10.4.10.254/24
!
interface Vlan20
   vrf RED
   ip address virtual 10.5.20.254/24
!
interface Vlan4090
   description MLAG Peer-address
   ip address 10.199.250.2/30
!
interface Vlan4091
   description for_iBGP
   ip address 10.199.251.2/30
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
   vxlan vrf BLUE vni 11001
   vxlan vrf RED vni 11002
   vxlan learn-restrict any
!
ip virtual-router mac-address 00:00:00:00:00:01
!
   
```
</details>
<details>
<summary> Настройки маршрутизации BGP, Underlay и Overlay на DC1-Leaf-1-2 </summary>

```  
ip routing
ip routing vrf BLUE
ip routing vrf RED
!
ip prefix-list DIRECT_CONNECTED
   seq 10 permit 10.0.0.2/32
   seq 20 permit 10.1.0.2/32
!
mlag configuration
   domain-id mlag-999
   local-interface Vlan4090
   peer-address 10.199.250.1
   peer-link Port-Channel999
!
route-map RM_REDISTRIBUTE_CONNECTED permit 10
   match ip address prefix-list DIRECT_CONNECTED
!
router bgp 65101
   router-id 10.0.0.2
   timers bgp 3 9
   maximum-paths 4 ecmp 4
   neighbor OVERLAY peer group
   neighbor OVERLAY remote-as 65100
   neighbor OVERLAY update-source Loopback0
   neighbor OVERLAY ebgp-multihop 3
   neighbor OVERLAY send-community extended
   neighbor UNDERLAY peer group
   neighbor UNDERLAY remote-as 65100
   neighbor UNDERLAY password 7 4vMwMsSftvmv7pRrLIgVxA==
   neighbor UNDERLAY send-community
   neighbor 10.0.1.0 peer group OVERLAY
   neighbor 10.0.2.0 peer group OVERLAY
   neighbor 10.2.1.2 peer group UNDERLAY
   neighbor 10.2.2.2 peer group UNDERLAY
   neighbor 10.199.251.1 remote-as 65101
   neighbor 10.199.251.1 next-hop-self
   redistribute connected route-map RM_REDISTRIBUTE_CONNECTED
   !
   vlan 10
      rd 65101:10010
      route-target both 10:10010
      redistribute learned
   !
   vlan 20
      rd 65101:10020
      route-target both 20:10020
      redistribute learned
   !
   address-family evpn
      neighbor OVERLAY activate
   !
   address-family ipv4
      no neighbor OVERLAY activate
      neighbor 10.199.251.1 activate
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
end
```
</details>

[Полная конфигурация DC1-Leaf-1-2](https://github.com/egorvshch/DC-networks-design/blob/main/project/configs/DC1-Leaf-1-2)

</details>

<details>
<summary> 3.5 Настройка DC1-Leaf-2-1 </summary>
-

<details>
<summary> Базовые настройки и настройки интерфейсов на DC1-Leaf-2-1 </summary>

```  

service routing protocols model multi-agent
!
hostname DC1-Leaf-2-1
!
spanning-tree mode mstp
no spanning-tree vlan-id 4090-4091
!
vlan 10
   name service-10
!
vlan 20
   name service-20
!
vlan 4090
   name mlag-peer
   trunk group mlag-peer
!
vlan 4091
   name mlag-iBGP
   trunk group mlag-peer
!
vrf instance BLUE
!
vrf instance RED
!
interface Port-Channel10
   description to Host-3
   switchport trunk allowed vlan 10
   switchport mode trunk
   mlag 10
!
interface Port-Channel20
   description to Host-4
   switchport trunk allowed vlan 20
   switchport mode trunk
   mlag 20
!
interface Port-Channel999
   description mlag_peer-link_to_Leaf-2-2
   switchport mode trunk
   switchport trunk group mlag-peer
!
interface Ethernet1
   description to DC1-Spine-1
   no switchport
   ip address 10.2.1.5/31
!
interface Ethernet2
   description to DC1-Spine-2
   no switchport
   ip address 10.2.2.5/31
!
interface Ethernet3
   description mlag_peer-link_to_Leaf-1-2_Eth4
   switchport trunk allowed vlan 10
   switchport mode trunk
   channel-group 999 mode active
!
interface Ethernet4
   description mlag_peer-link_to_Leaf-1-2_Eth5
   switchport trunk allowed vlan 20
   switchport mode trunk
   channel-group 999 mode active
!
interface Ethernet5
   description Host-3_eth0/0
   switchport trunk allowed vlan 10
   switchport mode trunk
   channel-group 10 mode active
!
interface Ethernet6
   description Host-4_eth0/0
   switchport trunk allowed vlan 20
   switchport mode trunk
   channel-group 20 mode active
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
   ip address virtual 10.4.10.254/24
!
interface Vlan20
   vrf RED
   ip address virtual 10.5.20.254/24
!
interface Vlan4090
   description MLAG Peer-address
   ip address 10.199.250.5/30
!
interface Vlan4091
   description for_iBGP
   ip address 10.199.251.5/30
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
   vxlan vrf BLUE vni 11001
   vxlan vrf RED vni 11002
   vxlan learn-restrict any
!
ip virtual-router mac-address 00:00:00:00:00:01
!
```
</details>

<details>
<summary> Настройки маршрутизации BGP, Underlay и Overlay на DC1-Leaf-2-1 </summary>

```  
ip routing
ip routing vrf BLUE
ip routing vrf RED
!
ip prefix-list DIRECT_CONNECTED
   seq 10 permit 10.0.0.3/32
   seq 20 permit 10.1.0.3/32
!
mlag configuration
   domain-id mlag-999
   local-interface Vlan4090
   peer-address 10.199.250.6
   peer-link Port-Channel999
!
route-map RM_REDISTRIBUTE_CONNECTED permit 10
   match ip address prefix-list DIRECT_CONNECTED
!
router bgp 65102
   router-id 10.0.0.3
   timers bgp 3 9
   maximum-paths 4 ecmp 4
   neighbor OVERLAY peer group
   neighbor OVERLAY remote-as 65100
   neighbor OVERLAY update-source Loopback0
   neighbor OVERLAY ebgp-multihop 3
   neighbor OVERLAY send-community extended
   neighbor UNDERLAY peer group
   neighbor UNDERLAY remote-as 65100
   neighbor UNDERLAY password 7 4vMwMsSftvmv7pRrLIgVxA==
   neighbor UNDERLAY send-community
   neighbor 10.0.1.0 peer group OVERLAY
   neighbor 10.0.2.0 peer group OVERLAY
   neighbor 10.2.1.4 peer group UNDERLAY
   neighbor 10.2.2.4 peer group UNDERLAY
   neighbor 10.199.251.6 remote-as 65102
   neighbor 10.199.251.6 next-hop-self
   redistribute connected route-map RM_REDISTRIBUTE_CONNECTED
   !
   vlan 10
      rd 65102:10010
      route-target both 10:10010
      redistribute learned
   !
   vlan 20
      rd 65102:10020
      route-target both 20:10020
      redistribute learned
   !
   address-family evpn
      neighbor OVERLAY activate
   !
   address-family ipv4
      no neighbor OVERLAY activate
      neighbor 10.199.251.6 activate
   !
   vrf BLUE
      rd 10.1.0.3:1
      route-target import evpn 1:10001
      route-target export evpn 1:10001
      redistribute connected
   !
   vrf RED
      rd 10.1.0.3:2
      route-target import evpn 2:10002
      route-target export evpn 2:10002
      redistribute connected
!
end
```
</details>

[Полная конфигурация DC1-Leaf-2-1](https://github.com/egorvshch/DC-networks-design/blob/main/project/configs/DC1-Leaf-2-1)
</details>

<details>
<summary> 3.6 Настройка DC1-Leaf-2-2 </summary>
-

<details>
<summary> Базовые настройки и настройки интерфейсов на DC1-Leaf-1-2 </summary>

```  
!
service routing protocols model multi-agent
!
hostname DC1-Leaf-2-2
!
spanning-tree mode mstp
no spanning-tree vlan-id 4090-4091
!
vlan 10
   name service-10
!
vlan 20
   name service-20
!
vlan 4090
   name mlag-peer
   trunk group mlag-peer
!
vlan 4091
   name mlag-iBGP
   trunk group mlag-peer
!
vrf instance BLUE
!
vrf instance RED
!
interface Port-Channel10
   description to Host-3
   switchport trunk allowed vlan 10
   switchport mode trunk
   mlag 10
!
interface Port-Channel20
   description to Host-4
   switchport trunk allowed vlan 20
   switchport mode trunk
   mlag 20
!
interface Port-Channel999
   description mlag_peer-link_to_Leaf-2-2
   switchport mode trunk
   switchport trunk group mlag-peer
!
interface Ethernet1
   description to DC1-Spine-1
   no switchport
   ip address 10.2.1.7/31
!
interface Ethernet2
   description to DC1-Spine-2
   no switchport
   ip address 10.2.2.7/31
!
interface Ethernet3
   description mlag_peer-link_to_Leaf-2-1_Eth3
   channel-group 999 mode active
!
interface Ethernet4
   description mlag_peer-link_to_Leaf-1-2_Eth4
   channel-group 999 mode active
!
interface Ethernet5
   description Host-3_eth0/0
   switchport trunk allowed vlan 10
   switchport mode trunk
   channel-group 10 mode active
!
interface Ethernet6
   description Host-4_eth0/0
   switchport trunk allowed vlan 20
   switchport mode trunk
   channel-group 20 mode active
!
interface Loopback0
   ip address 10.0.0.4/32
!
interface Loopback1
   ip address 10.1.0.4/32
!
interface Vlan10
   vrf BLUE
   ip address virtual 10.4.10.254/24
!
interface Vlan20
   vrf RED
   ip address virtual 10.5.20.254/24
!
interface Vlan4090
   description MLAG Peer-address
   ip address 10.199.250.6/30
!
interface Vlan4091
   description for_iBGP
   ip address 10.199.251.6/30
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
   vxlan vrf BLUE vni 11001
   vxlan vrf RED vni 11002
   vxlan learn-restrict any
!
ip virtual-router mac-address 00:00:00:00:00:01
!
   
```
</details>
<details>
<summary> Настройки маршрутизации BGP, Underlay и Overlay на DC1-Leaf-2-2 </summary>

```  

ip routing
ip routing vrf BLUE
ip routing vrf RED
!
ip prefix-list DIRECT_CONNECTED
   seq 10 permit 10.0.0.4/32
   seq 20 permit 10.1.0.4/32
!
mlag configuration
   domain-id mlag-999
   local-interface Vlan4090
   peer-address 10.199.250.5
   peer-link Port-Channel999
!
route-map RM_REDISTRIBUTE_CONNECTED permit 10
   match ip address prefix-list DIRECT_CONNECTED
!
router bgp 65102
   router-id 10.0.0.4
   timers bgp 3 9
   maximum-paths 4 ecmp 4
   neighbor OVERLAY peer group
   neighbor OVERLAY remote-as 65100
   neighbor OVERLAY update-source Loopback0
   neighbor OVERLAY ebgp-multihop 3
   neighbor OVERLAY send-community extended
   neighbor UNDERLAY peer group
   neighbor UNDERLAY remote-as 65100
   neighbor UNDERLAY password 7 4vMwMsSftvmv7pRrLIgVxA==
   neighbor UNDERLAY send-community
   neighbor 10.0.1.0 peer group OVERLAY
   neighbor 10.0.2.0 peer group OVERLAY
   neighbor 10.2.1.6 peer group UNDERLAY
   neighbor 10.2.2.6 peer group UNDERLAY
   neighbor 10.199.251.5 remote-as 65102
   neighbor 10.199.251.5 next-hop-self
   redistribute connected route-map RM_REDISTRIBUTE_CONNECTED
   !
   vlan 10
      rd 65102:10010
      route-target both 10:10010
      redistribute learned
   !
   vlan 20
      rd 65102:10020
      route-target both 20:10020
      redistribute learned
   !
   address-family evpn
      neighbor OVERLAY activate
   !
   address-family ipv4
      no neighbor OVERLAY activate
      neighbor 10.199.251.5 activate
   !
   vrf BLUE
      rd 10.1.0.4:1
      route-target import evpn 1:10001
      route-target export evpn 1:10001
      redistribute connected
   !
   vrf RED
      rd 10.1.0.4:2
      route-target import evpn 2:10002
      route-target export evpn 2:10002
      redistribute connected
!
end
```
</details>

[Полная конфигурация DC1-Leaf-2-2](https://github.com/egorvshch/DC-networks-design/blob/main/project/configs/DC1-Leaf-2-2)
</details>

<details>
<summary> 3.7 Настройка DC1-Border-GW1 </summary>
-
<details>
<summary> Базовые настройки и настройки интерфейсов на DC1-Border-GW1 </summary>

```  

service routing protocols model multi-agent
!
hostname DC1-Border-GW1
!
vlan 10
   name service-10
!
vrf instance BLUE
vrf instance GREEN
vrf instance RED
!
interface Ethernet1
   description to DC1-Spine-1
   no switchport
   ip address 10.2.1.9/31
!
interface Ethernet2
   description to DC1-Spine-2
   no switchport
   ip address 10.2.2.9/31
!
interface Ethernet3
   description to DC1-Border-R1
   no switchport
!
interface Ethernet3.104
   encapsulation dot1q vlan 104
   vrf BLUE
   ip address 10.198.104.1/30
!
interface Ethernet3.105
   encapsulation dot1q vlan 105
   vrf RED
   ip address 10.198.105.1/30
!
interface Ethernet3.106
   encapsulation dot1q vlan 106
   vrf GREEN
   ip address 10.198.106.1/30
!
interface Ethernet4
   description to_DCI-Router-1
   no switchport
   ip address 10.2.91.1/31
!
interface Ethernet5
   description to_DCI-Router-2
   no switchport
   ip address 10.2.92.1/31
!
interface Ethernet6
   description to DC2-Border-R1
   no switchport
!
interface Ethernet6.114
   encapsulation dot1q vlan 114
   vrf BLUE
   ip address 10.198.104.9/30
!
interface Ethernet6.115
   encapsulation dot1q vlan 115
   vrf RED
   ip address 10.198.105.9/30
!
interface Ethernet6.116
   encapsulation dot1q vlan 116
   vrf GREEN
   ip address 10.198.106.9/30
!
interface Loopback0
   ip address 10.0.0.5/32
!
interface Loopback1
   ip address 10.1.0.5/32
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan virtual-router encapsulation mac-address 00:00:00:10:10:11
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vrf BLUE vni 11001
   vxlan vrf RED vni 11002
   vxlan learn-restrict any
!
ip virtual-router mac-address 00:00:00:00:00:01
!
   
```
</details>

<details>
<summary> Настройки маршрутизации BGP, nderlay и Overlay на DC1-Border-GW1 </summary>

```  
ip routing
ip routing vrf BLUE
ip routing vrf GREEN
ip routing vrf RED
!
ip prefix-list DIRECT_CONNECTED
   seq 10 permit 10.0.0.5/32
   seq 20 permit 10.1.0.5/32
!
route-map LOCAL_DIRECT_CONNECTED permit 10
   match ip address prefix-list DIRECT_CONNECTED
   match source-protocol connected
!
route-map RM_REDISTRIBUTE_CONNECTED permit 10
   match ip address prefix-list DIRECT_CONNECTED
!
router bgp 65198
   router-id 10.0.0.5
   timers bgp 3 9
   maximum-paths 4 ecmp 4
   neighbor DCI-OVERLAY peer group
   neighbor DCI-OVERLAY remote-as 65999
   neighbor DCI-OVERLAY update-source Loopback0
   neighbor DCI-OVERLAY ebgp-multihop 3
   neighbor DCI-OVERLAY send-community extended
   neighbor DCI-UNDERLAY peer group
   neighbor DCI-UNDERLAY remote-as 65999
   neighbor DCI-UNDERLAY route-map LOCAL_DIRECT_CONNECTED out
   neighbor DCI-UNDERLAY password 7 gyEXkFgefQGXVmT5Pf3u4Q==
   neighbor DCI-UNDERLAY send-community
   neighbor OVERLAY peer group
   neighbor OVERLAY remote-as 65100
   neighbor OVERLAY update-source Loopback0
   neighbor OVERLAY ebgp-multihop 3
   neighbor OVERLAY send-community extended
   neighbor UNDERLAY peer group
   neighbor UNDERLAY remote-as 65100
   neighbor UNDERLAY route-map LOCAL_DIRECT_CONNECTED out
   neighbor UNDERLAY password 7 4vMwMsSftvmv7pRrLIgVxA==
   neighbor UNDERLAY send-community
   neighbor 10.0.1.0 peer group OVERLAY
   neighbor 10.0.2.0 peer group OVERLAY
   neighbor 10.2.1.8 peer group UNDERLAY
   neighbor 10.2.2.8 peer group UNDERLAY
   neighbor 10.2.91.0 peer group DCI-UNDERLAY
   neighbor 10.2.92.0 peer group DCI-UNDERLAY
   neighbor 10.91.91.1 peer group DCI-OVERLAY
   neighbor 10.91.91.2 peer group DCI-OVERLAY
   redistribute connected route-map RM_REDISTRIBUTE_CONNECTED
   !
   vlan 10
      rd evpn domain all 10.0.0.5:20010
      route-target both 10:10010
      route-target import export evpn domain all 99999:10
      redistribute learned
   !
   address-family evpn
      neighbor DCI-OVERLAY activate
      neighbor DCI-OVERLAY domain remote
      neighbor OVERLAY activate
      neighbor default next-hop-self received-evpn-routes route-type ip-prefix inter-domain
   !
   address-family ipv4
      no neighbor DCI-OVERLAY activate
      no neighbor OVERLAY activate
   !
   vrf BLUE
      rd 10.1.0.5:1
      route-target import evpn 1:10001
      route-target export evpn 1:10001
      neighbor 10.198.104.2 remote-as 65301
      neighbor 10.198.104.2 password 7 nHDSggPRmZnhQOoJn+6Bog==
      neighbor 10.198.104.10 remote-as 65301
      neighbor 10.198.104.10 password 7 nuL8/+vpAUGmKmauuRqggQ==
      aggregate-address 10.4.0.0/16 summary-only advertise-only
      aggregate-address 10.14.0.0/16 summary-only advertise-only
   !
   vrf GREEN
      rd 10.1.0.5:3
      route-target import evpn 3:10003
      route-target export evpn 3:10003
      neighbor 10.198.106.2 remote-as 65301
      neighbor 10.198.106.2 password 7 Wb/uzWsd1lVY/22MLmr58A==
      neighbor 10.198.106.10 remote-as 65301
      neighbor 10.198.106.10 password 7 Q9PVeHPb56JNoZ9QAiCP8Q==
      aggregate-address 10.6.0.0/16 summary-only advertise-only
      aggregate-address 10.13.0.0/16 summary-only advertise-only
   !
   vrf RED
      rd 10.1.0.5:2
      route-target import evpn 2:10002
      route-target export evpn 2:10002
      neighbor 10.198.105.2 remote-as 65301
      neighbor 10.198.105.2 password 7 nHDSggPRmZnhQOoJn+6Bog==
      neighbor 10.198.105.10 remote-as 65301
      neighbor 10.198.105.10 password 7 nuL8/+vpAUGmKmauuRqggQ==
      aggregate-address 10.5.0.0/16 summary-only advertise-only
      aggregate-address 10.12.0.0/16 summary-only advertise-only
!
```
</details>

[Полная конфигурация DC1-Border-GW1](https://github.com/egorvshch/DC-networks-design/blob/main/project/configs/DC1-Border-GW1)
</details>

<details>
<summary> 3.8 Настройка DC1-Border-GW2 </summary>
-
<details>
<summary> Базовые настройки и настройки интерфейсов на DC1-Border-GW2 </summary>

```  

service routing protocols model multi-agent
!
hostname DC1-Border-GW2
!
vlan 10
   name service-10
!
vrf instance BLUE
vrf instance GREEN
vrf instance RED
!
interface Ethernet1
   description to DC1-Spine-1
   no switchport
   ip address 10.2.1.11/31
!
interface Ethernet2
   description to DC1-Spine-2
   no switchport
   ip address 10.2.2.11/31
!
interface Ethernet3
   description to DC1-Border-R1
   no switchport
!
interface Ethernet3.104
   encapsulation dot1q vlan 104
   vrf BLUE
   ip address 10.199.104.1/30
!
interface Ethernet3.105
   encapsulation dot1q vlan 105
   vrf RED
   ip address 10.199.105.1/30
!
interface Ethernet3.106
   encapsulation dot1q vlan 106
   vrf GREEN
   ip address 10.199.106.1/30
!
interface Ethernet4
   description to_DCI-Router-1
   no switchport
   ip address 10.2.91.3/31
!
interface Ethernet5
   description to_DCI-Router-2
   no switchport
   ip address 10.2.92.3/31
!
interface Ethernet6
   description to DC2-Border-R1
   no switchport
!
interface Ethernet6.114
   encapsulation dot1q vlan 114
   vrf BLUE
   ip address 10.199.104.9/30
!
interface Ethernet6.115
   encapsulation dot1q vlan 115
   vrf RED
   ip address 10.199.105.9/30
!
interface Ethernet6.116
   encapsulation dot1q vlan 116
   vrf GREEN
   ip address 10.199.106.9/30
!
interface Loopback0
   ip address 10.0.0.6/32
!
interface Loopback1
   ip address 10.1.0.5/32
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan virtual-router encapsulation mac-address 00:00:00:10:10:11
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vrf BLUE vni 11001
   vxlan vrf RED vni 11002
   vxlan learn-restrict any
!
ip virtual-router mac-address 00:00:00:00:00:01
!
   
```
</details>

<details>
<summary> Настройки маршрутизации BGP, nderlay и Overlay на DC1-Border-GW2 </summary>

```  
ip routing
ip routing vrf BLUE
ip routing vrf GREEN
ip routing vrf RED
!
ip prefix-list DIRECT_CONNECTED
   seq 10 permit 10.0.0.6/32
   seq 20 permit 10.1.0.6/32
   seq 30 permit 10.1.0.5/32
!
route-map LOCAL_DIRECT_CONNECTED permit 10
   match ip address prefix-list DIRECT_CONNECTED
   match source-protocol connected
!
route-map RM_REDISTRIBUTE_CONNECTED permit 10
   match ip address prefix-list DIRECT_CONNECTED
!
router bgp 65199
   router-id 10.0.0.6
   timers bgp 3 9
   maximum-paths 4 ecmp 4
   neighbor DCI-OVERLAY peer group
   neighbor DCI-OVERLAY remote-as 65999
   neighbor DCI-OVERLAY update-source Loopback0
   neighbor DCI-OVERLAY ebgp-multihop 3
   neighbor DCI-OVERLAY send-community extended
   neighbor DCI-UNDERLAY peer group
   neighbor DCI-UNDERLAY remote-as 65999
   neighbor DCI-UNDERLAY route-map LOCAL_DIRECT_CONNECTED out
   neighbor DCI-UNDERLAY password 7 gyEXkFgefQGXVmT5Pf3u4Q==
   neighbor DCI-UNDERLAY send-community
   neighbor OVERLAY peer group
   neighbor OVERLAY remote-as 65100
   neighbor OVERLAY update-source Loopback0
   neighbor OVERLAY ebgp-multihop 3
   neighbor OVERLAY send-community extended
   neighbor UNDERLAY peer group
   neighbor UNDERLAY remote-as 65100
   neighbor UNDERLAY route-map LOCAL_DIRECT_CONNECTED out
   neighbor UNDERLAY password 7 4vMwMsSftvmv7pRrLIgVxA==
   neighbor UNDERLAY send-community
   neighbor 10.0.1.0 peer group OVERLAY
   neighbor 10.0.2.0 peer group OVERLAY
   neighbor 10.2.1.10 peer group UNDERLAY
   neighbor 10.2.2.10 peer group UNDERLAY
   neighbor 10.2.91.2 peer group DCI-UNDERLAY
   neighbor 10.2.92.2 peer group DCI-UNDERLAY
   neighbor 10.91.91.1 peer group DCI-OVERLAY
   neighbor 10.91.91.2 peer group DCI-OVERLAY
   redistribute connected route-map RM_REDISTRIBUTE_CONNECTED
   !
   vlan 10
      rd evpn domain all 10.0.0.6:20010
      route-target both 10:10010
      route-target import export evpn domain all 99999:10
      redistribute learned
   !
   address-family evpn
      neighbor DCI-OVERLAY activate
      neighbor DCI-OVERLAY domain remote
      neighbor OVERLAY activate
      neighbor default next-hop-self received-evpn-routes route-type ip-prefix inter-domain
   !
   address-family ipv4
      no neighbor DCI-OVERLAY activate
      no neighbor OVERLAY activate
   !
   vrf BLUE
      rd 10.1.0.6:1
      route-target import evpn 1:10001
      route-target export evpn 1:10001
      neighbor 10.199.104.2 remote-as 65301
      neighbor 10.199.104.2 password 7 nHDSggPRmZnhQOoJn+6Bog==
      neighbor 10.199.104.10 remote-as 65301
      neighbor 10.199.104.10 password 7 nuL8/+vpAUGmKmauuRqggQ==
      aggregate-address 10.4.0.0/16 summary-only advertise-only
      aggregate-address 10.14.0.0/16 summary-only advertise-only
   !
   vrf GREEN
      rd 10.1.0.6:3
      route-target import evpn 3:10003
      route-target export evpn 3:10003
      neighbor 10.199.106.2 remote-as 65301
      neighbor 10.199.106.2 password 7 Wb/uzWsd1lVY/22MLmr58A==
      neighbor 10.199.106.10 remote-as 65301
      neighbor 10.199.106.10 password 7 Q9PVeHPb56JNoZ9QAiCP8Q==
      aggregate-address 10.6.0.0/16 summary-only advertise-only
      aggregate-address 10.13.0.0/16 summary-only advertise-only
   !
   vrf RED
      rd 10.1.0.6:2
      route-target import evpn 2:10002
      route-target export evpn 2:10002
      neighbor 10.199.105.2 remote-as 65301
      neighbor 10.199.105.2 password 7 nHDSggPRmZnhQOoJn+6Bog==
      neighbor 10.199.105.10 remote-as 65301
      neighbor 10.199.105.10 password 7 nuL8/+vpAUGmKmauuRqggQ==
      aggregate-address 10.5.0.0/16 summary-only advertise-only
      aggregate-address 10.12.0.0/16 summary-only advertise-only
!
```
</details>

[Полная конфигурация DC1-Border-GW2](https://github.com/egorvshch/DC-networks-design/blob/main/project/configs/DC1-Border-GW2)
</details>

<details>
<summary> 3.9 Настройка DC1-Border-R1 </summary>
-
<details>
<summary> Базовые настройки и настройки интерфейсов на DC1-Border-R1 </summary>

```  
service routing protocols model multi-agent
!
hostname DC1-Border-R1
!
interface Ethernet1
   description to_DC1-Border-GW1
   no switchport
!
interface Ethernet1.104
   description to_vrf_BLUE_for_DC1-Border-GW1
   encapsulation dot1q vlan 104
   ip address 10.198.104.2/30
!
interface Ethernet1.105
   description to_vrf_RED_for_DC1-Border-GW1
   encapsulation dot1q vlan 105
   ip address 10.198.105.2/30
!
interface Ethernet1.106
   description to_vrf_GREEN_for_DC1-Border-GW1
   encapsulation dot1q vlan 106
   ip address 10.198.106.2/30
!
interface Ethernet2
   description to_DC1-Border-GW2
   no switchport
!
interface Ethernet2.104
   description to_vrf_BLUE_for_DC1-Border-GW2
   encapsulation dot1q vlan 104
   ip address 10.199.104.2/30
!
interface Ethernet2.105
   description to_vrf_RED_for_DC1-Border-GW2
   encapsulation dot1q vlan 105
   ip address 10.199.105.2/30
!
interface Ethernet2.106
   description to_vrf_GREEN_for_DC1-Border-GW2
   encapsulation dot1q vlan 106
   ip address 10.199.106.2/30
!
interface Ethernet3
   description to_DC2-Border-GW1
   no switchport
!
interface Ethernet3.114
   description to_vrf_BLUE_for_DC2-Border-GW1
   encapsulation dot1q vlan 114
   ip address 10.198.104.14/30
!
interface Ethernet3.115
   description to_vrf_RED_for_DC2-Border-GW1
   encapsulation dot1q vlan 115
   ip address 10.198.105.14/30
!
interface Ethernet3.116
   description to_vrf_GREEN_for_DC2-Border-GW1
   encapsulation dot1q vlan 116
   ip address 10.198.106.14/30
!
interface Ethernet4
   description to_DC1-Border-GW2
   no switchport
!
interface Ethernet4.114
   description to_vrf_BLUE_for_DC2-Border-GW2
   encapsulation dot1q vlan 114
   ip address 10.199.104.14/30
!
interface Ethernet4.115
   description to_vrf_RED_for_DC2-Border-GW2
   encapsulation dot1q vlan 115
   ip address 10.199.105.14/30
!
interface Ethernet4.116
   description to_vrf_GREEN_for_DC2-Border-GW2
   encapsulation dot1q vlan 116
   ip address 10.199.106.14/30
!
interface Ethernet5
   description to DC2-Border-R1
   no switchport
   ip address 10.195.0.1/30
!
interface Loopback0
   ip address 10.199.0.1/32
!
interface Loopback11
   ip address 1.1.1.1/24
!

```
</details>
<details>
<summary> Настройки маршрутизации BGP, Underlay и Overlay на DC1-Border-R1 </summary>

```  
ip routing
!
ip prefix-list LOOPBACKs
   seq 10 permit 1.1.1.0/24
!
ip prefix-list for_DC1-Border-GW
   seq 10 permit 0.0.0.0/0
!
route-map RM-REDISTRIB-CONNECTED permit 10
   match ip address prefix-list LOOPBACKs
!
route-map for_DC1-Border-GW_out permit 10
   match ip address prefix-list for_DC1-Border-GW
!
router bgp 65301
   bgp asn notation asdot
   router-id 10.199.0.1
   timers bgp 3 9
   maximum-paths 24
   neighbor DC1-Border-GW1 peer group
   neighbor DC1-Border-GW1 remote-as 65198
   neighbor DC1-Border-GW1 route-map for_DC1-Border-GW_out out
   neighbor DC1-Border-GW1 password 7 mXpeujoAlay5K6JXiQjWDQ==
   neighbor DC1-Border-GW1 default-originate
   neighbor DC1-Border-GW2 peer group
   neighbor DC1-Border-GW2 remote-as 65199
   neighbor DC1-Border-GW2 route-map for_DC1-Border-GW_out out
   neighbor DC1-Border-GW2 password 7 mvBH1FR+M+6kO9JylsFXRg==
   neighbor DC1-Border-GW2 default-originate
   neighbor DC2-Border-GW1 peer group
   neighbor DC2-Border-GW1 remote-as 65298
   neighbor DC2-Border-GW1 route-map for_DC1-Border-GW_out out
   neighbor DC2-Border-GW1 password 7 Jd5ObGtFDc7qf/cuFnrqTA==
   neighbor DC2-Border-GW1 default-originate
   neighbor DC2-Border-GW2 peer group
   neighbor DC2-Border-GW2 remote-as 65299
   neighbor DC2-Border-GW2 route-map for_DC1-Border-GW_out out
   neighbor DC2-Border-GW2 password 7 b9rsJJT1YnrVQNUUBHZtKw==
   neighbor DC2-Border-GW2 default-originate
   neighbor 10.195.0.2 remote-as 65301
   neighbor 10.195.0.2 next-hop-self
   neighbor 10.195.0.2 password 7 FQXiOIsZVLw/KGDnZBvVJQ==
   neighbor 10.198.104.1 peer group DC1-Border-GW1
   neighbor 10.198.104.13 peer group DC2-Border-GW1
   neighbor 10.198.105.1 peer group DC1-Border-GW1
   neighbor 10.198.105.13 peer group DC2-Border-GW1
   neighbor 10.198.106.1 peer group DC1-Border-GW1
   neighbor 10.198.106.13 peer group DC2-Border-GW1
   neighbor 10.199.104.1 peer group DC1-Border-GW2
   neighbor 10.199.104.13 peer group DC2-Border-GW2
   neighbor 10.199.105.1 peer group DC1-Border-GW2
   neighbor 10.199.105.13 peer group DC2-Border-GW2
   neighbor 10.199.106.1 peer group DC1-Border-GW2
   neighbor 10.199.106.13 peer group DC2-Border-GW2
   redistribute connected route-map RM-REDISTRIB-CONNECTED
!
```
</details>

[Полная конфигурация DC1-Border-R1](https://github.com/egorvshch/DC-networks-design/blob/main/project/configs/DC1-Border-R1)
</details>

<details>
<summary> 3.10 Настройка DC1-Host-1, DC1-Host-2, DC1-Host-3 и DC1-Host-4 </summary>
-
конфигурация хостов указана по сылками ниже:

[Полная конфигурация DC1-Host-1](https://github.com/egorvshch/DC-networks-design/blob/main/project/configs/DC1-Host-1)

[Полная конфигурация DC1-Host-2](https://github.com/egorvshch/DC-networks-design/blob/main/project/configs/DC1-Host-2)

[Полная конфигурация DC1-Host-3](https://github.com/egorvshch/DC-networks-design/blob/main/project/configs/DC1-Host-3)

[Полная конфигурация DC1-Host-4](https://github.com/egorvshch/DC-networks-design/blob/main/project/configs/DC1-Host-4)
</details>

### 4. Настройка оборудрования DC2:

<details>
<summary> 4.1 Настройка DC2-Spine-1 </summary>
-
<details>
<summary> Базовые настройки и настройки интерфейсов на DC2-Spine-1 </summary>

```  
service routing protocols model multi-agent
!
hostname DC2-Spine-1
!
interface Ethernet1
   description to DC2-Leaf-1-1
   no switchport
   ip address 10.10.1.0/31
!
interface Ethernet2
   description to DC2-Leaf-1-2
   no switchport
   ip address 10.10.1.2/31
!
interface Ethernet3
   description to DC2-Leaf-2-1
   no switchport
   ip address 10.10.1.4/31
!
interface Ethernet4
   description to DC2-Leaf-2-2
   no switchport
   ip address 10.10.1.6/31
!
interface Ethernet5
   description to DC2-Border-GW1
   no switchport
   ip address 10.10.1.8/31
!
interface Ethernet6
   description to DC2-Border-GW2
   no switchport
   ip address 10.10.1.10/31
!
interface Loopback0
   ip address 10.8.1.0/32
!
interface Loopback1
   ip address 10.9.1.0/32
!
```
</details>
<details>
<summary> Настройки маршрутизации BGP, Underlay и Overlay на DC2-Spine-1 </summary>

```  

ip routing
!
ip prefix-list DIRECT_CONNECTED
   seq 10 permit 10.8.1.0/32
   seq 20 permit 10.9.1.0/32
!
route-map RM_REDISTRIBUTE_CONNECTED permit 10
   match ip address prefix-list DIRECT_CONNECTED
!
peer-filter OVERLAY
   10 match as-range 65201-65299 result accept
!
peer-filter UNDERLAY
   10 match as-range 65201-65299 result accept
!
router bgp 65200
   bgp asn notation asdot
   router-id 10.8.1.0
   timers bgp 3 9
   maximum-paths 4 ecmp 4
   bgp listen range 10.8.0.0/24 peer-group OVERLAY peer-filter OVERLAY
   bgp listen range 10.10.1.0/24 peer-group UNDERLAY peer-filter UNDERLAY
   neighbor OVERLAY peer group
   neighbor OVERLAY next-hop-unchanged
   neighbor OVERLAY update-source Loopback0
   neighbor OVERLAY ebgp-multihop 3
   neighbor OVERLAY send-community extended
   neighbor UNDERLAY peer group
   neighbor UNDERLAY password 7 4vMwMsSftvmv7pRrLIgVxA==
   neighbor UNDERLAY send-community
   redistribute connected route-map RM_REDISTRIBUTE_CONNECTED
   !
   address-family evpn
      neighbor OVERLAY activate
   !
   address-family ipv4
      no neighbor OVERLAY activate
!
end
```
</details>

[Полная конфигурация DC2-Spine-1](https://github.com/egorvshch/DC-networks-design/blob/main/project/configs/DC2-Spine-1)
</details>

<details>
<summary> 4.2 Настройка DC2-Spine-2 </summary>
-

<details>
<summary> Базовые настройки и настройки интерфейсов на DC2-Spine-2 </summary>

```  
service routing protocols model multi-agent
!
hostname DC2-Spine-2
!
interface Ethernet1
   description to DC2-Leaf-1-1
   no switchport
   ip address 10.10.2.0/31
!
interface Ethernet2
   description to DC2-Leaf-1-2
   no switchport
   ip address 10.10.2.2/31
!
interface Ethernet3
   description to DC2-Leaf-2-1
   no switchport
   ip address 10.10.2.4/31
!
interface Ethernet4
   description to DC2-Leaf-2-2
   no switchport
   ip address 10.10.2.6/31
!
interface Ethernet5
   description to DC2-Border-GW1
   no switchport
   ip address 10.10.2.8/31
!
interface Ethernet6
   description to DC2-Border-GW1
   no switchport
   ip address 10.10.2.10/31
!
interface Loopback0
   ip address 10.8.2.0/32
!
interface Loopback1
   ip address 10.9.2.0/32
!
   
```
</details>
<details>
<summary> Настройки маршрутизации BGP, Underlay и Overlay на DC2-Spine-2 </summary>

```  

ip routing
!
ip prefix-list DIRECT_CONNECTED
   seq 10 permit 10.8.2.0/32
   seq 20 permit 10.9.2.0/32
!
route-map RM_REDISTRIBUTE_CONNECTED permit 10
   match ip address prefix-list DIRECT_CONNECTED
!
peer-filter OVERLAY
   10 match as-range 65201-65299 result accept
!
peer-filter UNDERLAY
   10 match as-range 65201-65299 result accept
!
router bgp 65200
   bgp asn notation asdot
   router-id 10.8.2.0
   timers bgp 3 9
   maximum-paths 4 ecmp 4
   bgp listen range 10.8.0.0/24 peer-group OVERLAY peer-filter OVERLAY
   bgp listen range 10.10.2.0/24 peer-group UNDERLAY peer-filter UNDERLAY
   neighbor OVERLAY peer group
   neighbor OVERLAY next-hop-unchanged
   neighbor OVERLAY update-source Loopback0
   neighbor OVERLAY ebgp-multihop 3
   neighbor OVERLAY send-community extended
   neighbor UNDERLAY peer group
   neighbor UNDERLAY password 7 4vMwMsSftvmv7pRrLIgVxA==
   neighbor UNDERLAY send-community
   redistribute connected route-map RM_REDISTRIBUTE_CONNECTED
   !
   address-family evpn
      neighbor OVERLAY activate
   !
   address-family ipv4
      no neighbor OVERLAY activate
!
end
```
</details>

[Полная конфигурация DC2-Spine-2](https://github.com/egorvshch/DC-networks-design/blob/main/project/configs/DC2-Spine-2)
</details>

<details>
<summary> 4.3 Настройка DC2-Leaf-1-1 </summary>
-
<details>
<summary> Базовые настройки и настройки интерфейсов на DC2-Leaf-1-1 </summary>

```  
service routing protocols model multi-agent
!
hostname DC2-Leaf-1-1
!
spanning-tree mode mstp
no spanning-tree vlan-id 4090-4091
!
vlan 10
   name service-10
!
vlan 14
   name service-14
!
vlan 20
   name service-20
!
vlan 30
   name service-30
!
vlan 4090
   name mlag-peer
   trunk group mlag-peer
!
vlan 4091
   name mlag-iBGP
   trunk group mlag-peer
!
vrf instance BLUE
vrf instance GREEN
vrf instance RED
!
interface Port-Channel10
   description to DC2-Host-1
   switchport trunk allowed vlan 10
   switchport mode trunk
   mlag 10
!
interface Port-Channel20
   description to DC2-Host-2
   switchport trunk allowed vlan 20
   switchport mode trunk
   mlag 20
!
interface Port-Channel30
   description to DC2-Host-2
   switchport trunk allowed vlan 30
   switchport mode trunk
   mlag 30
!
interface Port-Channel999
   description mlag_peer-link_to_Leaf-1-2
   switchport mode trunk
   switchport trunk group mlag-peer
!
interface Ethernet1
   description to DC2-Spine-1
   no switchport
   ip address 10.10.1.1/31
!
interface Ethernet2
   description to DC2-Spine-2
   no switchport
   ip address 10.10.2.1/31
!
interface Ethernet3
   description mlag_peer-link_to_Leaf-1-2_Eth4
   switchport trunk allowed vlan 10
   switchport mode trunk
   channel-group 999 mode active
!
interface Ethernet4
   description mlag_peer-link_to_Leaf-1-2_Eth5
   switchport trunk allowed vlan 20
   switchport mode trunk
   channel-group 999 mode active
!
interface Ethernet5
   description DC2-Host-1_eth0/0
   switchport trunk allowed vlan 10
   switchport mode trunk
   channel-group 10 mode active
!
interface Ethernet6
   description DC2-Host-2_eth0/0
   switchport trunk allowed vlan 30
   switchport mode trunk
   channel-group 30 mode active
!
interface Loopback0
   ip address 10.8.0.1/32
!
interface Loopback1
   ip address 10.9.0.1/32
!
interface Management1
!
interface Vlan10
   vrf BLUE
   ip address virtual 10.4.10.254/24
!
interface Vlan14
   vrf BLUE
   ip address virtual 10.14.14.254/24
!
interface Vlan20
   vrf RED
   ip address virtual 10.12.20.254/24
!
interface Vlan30
   vrf GREEN
   ip address virtual 10.13.30.254/24
!
interface Vlan4090
   description MLAG Peer-address
   ip address 10.199.252.1/30
!
interface Vlan4091
   description for_iBGP
   ip address 10.199.253.1/30
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
   vxlan vlan 30 vni 10030
   vxlan vrf BLUE vni 11001
   vxlan vrf GREEN vni 11003
   vxlan vrf RED vni 11002
   vxlan learn-restrict any
!
ip virtual-router mac-address 00:00:00:00:00:01
!
```
</details>

<details>
<summary> Настройки маршрутизации BGP, Underlay и Overlay на DC2-Leaf-1-1 </summary>

```  
ip routing
ip routing vrf BLUE
ip routing vrf GREEN
ip routing vrf RED
!
ip prefix-list DIRECT_CONNECTED
   seq 10 permit 10.8.0.1/32
   seq 20 permit 10.9.0.1/32
!
mlag configuration
   domain-id mlag-999
   local-interface Vlan4090
   peer-address 10.199.252.2
   peer-link Port-Channel999
!
route-map RM_REDISTRIBUTE_CONNECTED permit 10
   match ip address prefix-list DIRECT_CONNECTED
!
router bgp 65201
   router-id 10.8.0.1
   timers bgp 3 9
   maximum-paths 4 ecmp 4
   neighbor OVERLAY peer group
   neighbor OVERLAY remote-as 65200
   neighbor OVERLAY update-source Loopback0
   neighbor OVERLAY ebgp-multihop 3
   neighbor OVERLAY send-community extended
   neighbor UNDERLAY peer group
   neighbor UNDERLAY remote-as 65200
   neighbor UNDERLAY password 7 4vMwMsSftvmv7pRrLIgVxA==
   neighbor UNDERLAY send-community
   neighbor 10.8.1.0 peer group OVERLAY
   neighbor 10.8.2.0 peer group OVERLAY
   neighbor 10.10.1.0 peer group UNDERLAY
   neighbor 10.10.2.0 peer group UNDERLAY
   neighbor 10.199.253.2 remote-as 65201
   neighbor 10.199.253.2 next-hop-self
   redistribute connected route-map RM_REDISTRIBUTE_CONNECTED
   !
   vlan 10
      rd 10.9.0.1:10010
      route-target both 10:10010
      redistribute learned
   !
   vlan 20
      rd 10.9.0.1:10020
      route-target both 20:10020
      redistribute learned
   !
   vlan 30
      rd 10.9.0.1:10030
      route-target both 30:10030
      redistribute learned
   !
   address-family evpn
      neighbor OVERLAY activate
   !
   address-family ipv4
      no neighbor OVERLAY activate
      neighbor 10.199.253.2 activate
   !
   vrf BLUE
      rd 10.9.0.1:1
      route-target import evpn 1:10001
      route-target export evpn 1:10001
      redistribute connected
   !
   vrf GREEN
      rd 10.9.0.1:3
      route-target import evpn 3:10003
      route-target export evpn 3:10003
      redistribute connected
   !
   vrf RED
      rd 10.9.0.1:2
      route-target import evpn 2:10002
      route-target export evpn 2:10002
      redistribute connected
!
end

```
</details>

[Полная конфигурация DC2-Leaf-1-1](https://github.com/egorvshch/DC-networks-design/blob/main/project/configs/DC2-Leaf-1-1)
</details>

<details>
<summary> 4.4 Настройка DC2-Leaf-1-2 </summary>
-
<details>
<summary> Базовые настройки и настройки интерфейсов на DC2-Leaf-1-2 </summary>

```  
!
service routing protocols model multi-agent
!
hostname DC2-Leaf-1-2
!
spanning-tree mode mstp
no spanning-tree vlan-id 4090-4091
!
vlan 10
   name service-10
!
vlan 14
   name service-14
!
vlan 20
   name service-20
!
vlan 30
   name service-30
!
vlan 4090
   name mlag-peer
   trunk group mlag-peer
!
vlan 4091
   name mlag-iBGP
   trunk group mlag-peer
!
vrf instance BLUE
vrf instance GREEN
vrf instance RED
!
interface Port-Channel10
   description to DC2-Host-1
   switchport trunk allowed vlan 10
   switchport mode trunk
   mlag 10
!
interface Port-Channel20
   description to DC2-Host-2
   switchport trunk allowed vlan 20
   switchport mode trunk
   mlag 20
!
interface Port-Channel30
   description to DC2-Host-2
   switchport trunk allowed vlan 30
   switchport mode trunk
   mlag 30
!
interface Port-Channel999
   description mlag_peer-link_to_Leaf-1-1
   switchport mode trunk
   switchport trunk group mlag-peer
!
interface Ethernet1
   description to DC2-Spine-1
   no switchport
   ip address 10.10.1.3/31
!
interface Ethernet2
   description to DC2-Spine-2
   no switchport
   ip address 10.10.2.3/31
!
interface Ethernet3
   description mlag_peer-link_to_DC2-Leaf-1-1_Eth4
   channel-group 999 mode active
!
interface Ethernet4
   description mlag_peer-link_to_DC2-Leaf-1-1_Eth5
   channel-group 999 mode active
!
interface Ethernet5
   description DC2-Host-1_eth0/1
   switchport trunk allowed vlan 10
   switchport mode trunk
   channel-group 10 mode active
!
interface Ethernet6
   description DC2-Host-2_eth0/1
   switchport trunk allowed vlan 30
   switchport mode trunk
   channel-group 30 mode active
!
interface Loopback0
   ip address 10.8.0.2/32
!
interface Loopback1
   ip address 10.9.0.2/32
!
interface Vlan10
   vrf BLUE
   ip address virtual 10.4.10.254/24
!
interface Vlan14
   vrf BLUE
   ip address virtual 10.14.14.254/24
!
interface Vlan20
   vrf RED
   ip address virtual 10.12.20.254/24
!
interface Vlan30
   vrf GREEN
   ip address virtual 10.13.30.254/24
!
interface Vlan4090
   description MLAG Peer-address
   ip address 10.199.252.2/30
!
interface Vlan4091
   description for_iBGP
   ip address 10.199.253.2/30
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
   vxlan vlan 30 vni 10030
   vxlan vrf BLUE vni 11001
   vxlan vrf GREEN vni 11003
   vxlan vrf RED vni 11002
   vxlan learn-restrict any
!
ip virtual-router mac-address 00:00:00:00:00:01
!
   
```
</details>
<details>
<summary> Настройки маршрутизации BGP, Underlay и Overlay на DC2-Leaf-1-2 </summary>

```  

ip routing
ip routing vrf BLUE
ip routing vrf GREEN
ip routing vrf RED
!
ip prefix-list DIRECT_CONNECTED
   seq 10 permit 10.8.0.2/32
   seq 20 permit 10.9.0.2/32
!
mlag configuration
   domain-id mlag-999
   local-interface Vlan4090
   peer-address 10.199.252.1
   peer-link Port-Channel999
!
route-map RM_REDISTRIBUTE_CONNECTED permit 10
   match ip address prefix-list DIRECT_CONNECTED
!
router bgp 65201
   router-id 10.8.0.2
   timers bgp 3 9
   maximum-paths 4 ecmp 4
   neighbor OVERLAY peer group
   neighbor OVERLAY remote-as 65200
   neighbor OVERLAY update-source Loopback0
   neighbor OVERLAY ebgp-multihop 3
   neighbor OVERLAY send-community extended
   neighbor UNDERLAY peer group
   neighbor UNDERLAY remote-as 65200
   neighbor UNDERLAY password 7 4vMwMsSftvmv7pRrLIgVxA==
   neighbor UNDERLAY send-community
   neighbor 10.8.1.0 peer group OVERLAY
   neighbor 10.8.2.0 peer group OVERLAY
   neighbor 10.10.1.2 peer group UNDERLAY
   neighbor 10.10.2.2 peer group UNDERLAY
   neighbor 10.199.253.1 remote-as 65201
   neighbor 10.199.253.1 next-hop-self
   redistribute connected route-map RM_REDISTRIBUTE_CONNECTED
   !
   vlan 10
      rd 10.9.0.2:10010
      route-target both 10:10010
      redistribute learned
   !
   vlan 20
      rd 10.9.0.2:10020
      route-target both 20:10020
      redistribute learned
   !
   vlan 30
      rd 10.9.0.2:10030
      route-target both 30:10030
      redistribute learned
   !
   address-family evpn
      neighbor OVERLAY activate
   !
   address-family ipv4
      no neighbor OVERLAY activate
      neighbor 10.199.253.1 activate
   !
   vrf BLUE
      rd 10.9.0.2:1
      route-target import evpn 1:10001
      route-target export evpn 1:10001
      redistribute connected
   !
   vrf GREEN
      rd 10.9.0.2:3
      route-target import evpn 3:10003
      route-target export evpn 3:10003
      redistribute connected
   !
   vrf RED
      rd 10.9.0.2:2
      route-target import evpn 2:10002
      route-target export evpn 2:10002
      redistribute connected
!
end
```
</details>

[Полная конфигурация DC2-Leaf-1-2](https://github.com/egorvshch/DC-networks-design/blob/main/project/configs/DC2-Leaf-1-2)
</details>

<details>
<summary> 4.5 Настройка DC2-Leaf-2-1 </summary>
-

<details>
<summary> Базовые настройки и настройки интерфейсов на DC2-Leaf-2-1 </summary>

```  
!
service routing protocols model multi-agent
!
hostname DC2-Leaf-2-1
!
spanning-tree mode mstp
no spanning-tree vlan-id 4090-4091
!
vlan 10
   name service-10
!
vlan 20
   name service-20
!
vlan 30
   name service-30
!
vlan 4090
   name mlag-peer
   trunk group mlag-peer
!
vlan 4091
   name mlag-iBGP
   trunk group mlag-peer
!
vrf instance BLUE
vrf instance GREEN
vrf instance RED
!
interface Port-Channel10
   description to DC2-Host-3
   switchport trunk allowed vlan 10
   switchport mode trunk
   mlag 10
!
interface Port-Channel20
   description to DC2-Host-4
   switchport trunk allowed vlan 20
   switchport mode trunk
   mlag 20
!
interface Port-Channel30
   description to DC2-Host-3
   switchport trunk allowed vlan 30
   switchport mode trunk
   mlag 30
!
interface Port-Channel999
   description mlag_peer-link_to_Leaf-2-2
   switchport mode trunk
   switchport trunk group mlag-peer
!
interface Ethernet1
   description to DC2-Spine-1
   no switchport
   ip address 10.10.1.5/31
!
interface Ethernet2
   description to DC2-Spine-2
   no switchport
   ip address 10.10.2.5/31
!
interface Ethernet3
   description mlag_peer-link_to_Leaf-1-2_Eth4
   switchport trunk allowed vlan 10
   switchport mode trunk
   channel-group 999 mode active
!
interface Ethernet4
   description mlag_peer-link_to_Leaf-1-2_Eth5
   switchport trunk allowed vlan 20
   switchport mode trunk
   channel-group 999 mode active
!
interface Ethernet5
   description DC2-Host-3_eth0/0
   switchport trunk allowed vlan 30
   switchport mode trunk
   channel-group 30 mode active
!
interface Ethernet6
   description DC2-Host-4_eth0/0
   switchport trunk allowed vlan 20
   switchport mode trunk
   channel-group 20 mode active
!
!
interface Loopback0
   ip address 10.8.0.3/32
!
interface Loopback1
   ip address 10.9.0.3/32
!
interface Vlan10
   vrf BLUE
   ip address virtual 10.4.10.254/24
!
interface Vlan20
   vrf RED
   ip address virtual 10.12.20.254/24
!
interface Vlan30
   vrf GREEN
   ip address virtual 10.13.30.254/24
!
interface Vlan4090
   description MLAG Peer-address
   ip address 10.199.252.5/30
!
interface Vlan4091
   description for_iBGP
   ip address 10.199.253.5/30
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
   vxlan vlan 30 vni 10030
   vxlan vrf BLUE vni 11001
   vxlan vrf GREEN vni 11003
   vxlan vrf RED vni 11002
   vxlan learn-restrict any
!
ip virtual-router mac-address 00:00:00:00:00:01
!
```
</details>

<details>
<summary> Настройки маршрутизации BGP, Underlay и Overlay на DC2-Leaf-2-1 </summary>

```  

ip routing
ip routing vrf BLUE
ip routing vrf GREEN
ip routing vrf RED
!
ip prefix-list DIRECT_CONNECTED
   seq 10 permit 10.8.0.3/32
   seq 20 permit 10.9.0.3/32
!
mlag configuration
   domain-id mlag-999
   local-interface Vlan4090
   peer-address 10.199.252.6
   peer-link Port-Channel999
!
route-map RM_REDISTRIBUTE_CONNECTED permit 10
   match ip address prefix-list DIRECT_CONNECTED
!
router bgp 65202
   router-id 10.8.0.3
   timers bgp 3 9
   maximum-paths 4 ecmp 4
   neighbor OVERLAY peer group
   neighbor OVERLAY remote-as 65200
   neighbor OVERLAY update-source Loopback0
   neighbor OVERLAY ebgp-multihop 3
   neighbor OVERLAY send-community extended
   neighbor UNDERLAY peer group
   neighbor UNDERLAY remote-as 65200
   neighbor UNDERLAY password 7 4vMwMsSftvmv7pRrLIgVxA==
   neighbor UNDERLAY send-community
   neighbor 10.8.1.0 peer group OVERLAY
   neighbor 10.8.2.0 peer group OVERLAY
   neighbor 10.10.1.4 peer group UNDERLAY
   neighbor 10.10.2.4 peer group UNDERLAY
   neighbor 10.199.253.6 remote-as 65202
   neighbor 10.199.253.6 next-hop-self
   redistribute connected route-map RM_REDISTRIBUTE_CONNECTED
   !
   vlan 10
      rd 10.9.0.3:10010
      route-target both 10:10010
      redistribute learned
   !
   vlan 20
      rd 10.9.0.3:10020
      route-target both 20:10020
      redistribute learned
   !
   vlan 30
      rd 10.9.0.3:10030
      route-target both 30:10030
      redistribute learned
   !
   address-family evpn
      neighbor OVERLAY activate
   !
   address-family ipv4
      no neighbor OVERLAY activate
      neighbor 10.199.253.6 activate
   !
   vrf BLUE
      rd 10.9.0.3:1
      route-target import evpn 1:10001
      route-target export evpn 1:10001
      redistribute connected
   !
   vrf GREEN
      rd 10.9.0.3:3
      route-target import evpn 3:10003
      route-target export evpn 3:10003
      redistribute connected
   !
   vrf RED
      rd 10.9.0.3:2
      route-target import evpn 2:10002
      route-target export evpn 2:10002
      redistribute connected
!
end
```
</details>

[Полная конфигурация DC2-Leaf-2-1](https://github.com/egorvshch/DC-networks-design/blob/main/project/configs/DC2-Leaf-2-1)
</details>

<details>
<summary> 4.6 Настройка DC2-Leaf-2-2 </summary>
-

<details>
<summary> Базовые настройки и настройки интерфейсов на DC2-Leaf-2-2 </summary>

```  
service routing protocols model multi-agent
!
hostname DC2-Leaf-2-2
!
spanning-tree mode mstp
no spanning-tree vlan-id 4090-4091
!
vlan 10
   name service-10
!
vlan 20
   name service-20
!
vlan 30
   name service-30
!
vlan 4090
   name mlag-peer
   trunk group mlag-peer
!
vlan 4091
   name mlag-iBGP
   trunk group mlag-peer
!
vrf instance BLUE
vrf instance GREEN
vrf instance RED
!
interface Port-Channel10
   description to DC2-Host-3
   switchport trunk allowed vlan 10
   switchport mode trunk
   mlag 10
!
interface Port-Channel20
   description to DC2-Host-4
   switchport trunk allowed vlan 20
   switchport mode trunk
   mlag 20
!
interface Port-Channel30
   description to DC2-Host-3
   switchport trunk allowed vlan 30
   switchport mode trunk
   mlag 30
!
interface Port-Channel999
   description mlag_peer-link_to_Leaf-2-1
   switchport mode trunk
   switchport trunk group mlag-peer
!
interface Ethernet1
   description to DC2-Spine-1
   no switchport
   ip address 10.10.1.7/31
!
interface Ethernet2
   description to DC2-Spine-2
   no switchport
   ip address 10.10.2.7/31
!
interface Ethernet3
   description mlag_peer-link_to_Leaf-2-1_Eth3
   channel-group 999 mode active
!
interface Ethernet4
   description mlag_peer-link_to_Leaf-2-1_Eth4
   channel-group 999 mode active
!
interface Ethernet5
   description DC2-Host-3_eth0/1
   switchport trunk allowed vlan 30
   switchport mode trunk
   channel-group 30 mode active
!
interface Ethernet6
   description DC2-Host-4_eth0/1
   switchport trunk allowed vlan 20
   switchport mode trunk
   channel-group 20 mode active
!
interface Loopback0
   ip address 10.8.0.4/32
!
interface Loopback1
   ip address 10.9.0.4/32
!
interface Vlan10
   vrf BLUE
   ip address virtual 10.4.10.254/24
!
interface Vlan20
   vrf RED
   ip address virtual 10.12.20.254/24
!
interface Vlan30
   vrf GREEN
   ip address virtual 10.13.30.254/24
!
interface Vlan4090
   description MLAG Peer-address
   ip address 10.199.252.6/30
!
interface Vlan4091
   description for_iBGP
   ip address 10.199.253.6/30
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
   vxlan vlan 30 vni 10030
   vxlan vrf BLUE vni 11001
   vxlan vrf GREEN vni 11003
   vxlan vrf RED vni 11002
   vxlan learn-restrict any
!
ip virtual-router mac-address 00:00:00:00:00:01
!
   
```
</details>
<details>
<summary> Настройки маршрутизации BGP, Underlay и Overlay на DC2-Leaf-2-2 </summary>

```  

ip routing
ip routing vrf BLUE
ip routing vrf GREEN
ip routing vrf RED
!
ip prefix-list DIRECT_CONNECTED
   seq 10 permit 10.8.0.4/32
   seq 20 permit 10.9.0.4/32
!
mlag configuration
   domain-id mlag-999
   local-interface Vlan4090
   peer-address 10.199.252.5
   peer-link Port-Channel999
!
route-map RM_REDISTRIBUTE_CONNECTED permit 10
   match ip address prefix-list DIRECT_CONNECTED
!
router bgp 65202
   router-id 10.8.0.4
   timers bgp 3 9
   maximum-paths 4 ecmp 4
   neighbor OVERLAY peer group
   neighbor OVERLAY remote-as 65200
   neighbor OVERLAY update-source Loopback0
   neighbor OVERLAY ebgp-multihop 3
   neighbor OVERLAY send-community extended
   neighbor UNDERLAY peer group
   neighbor UNDERLAY remote-as 65200
   neighbor UNDERLAY password 7 4vMwMsSftvmv7pRrLIgVxA==
   neighbor UNDERLAY send-community
   neighbor 10.8.1.0 peer group OVERLAY
   neighbor 10.8.2.0 peer group OVERLAY
   neighbor 10.10.1.6 peer group UNDERLAY
   neighbor 10.10.2.6 peer group UNDERLAY
   neighbor 10.199.253.5 remote-as 65202
   neighbor 10.199.253.5 next-hop-self
   redistribute connected route-map RM_REDISTRIBUTE_CONNECTED
   !
   vlan 10
      rd 10.9.0.4:10010
      route-target both 10:10010
      redistribute learned
   !
   vlan 20
      rd 10.9.0.4:10020
      route-target both 20:10020
      redistribute learned
   !
   vlan 30
      rd 10.9.0.4:10030
      route-target both 30:10030
      redistribute learned
   !
   address-family evpn
      neighbor OVERLAY activate
   !
   address-family ipv4
      no neighbor OVERLAY activate
      neighbor 10.199.253.5 activate
   !
   vrf BLUE
      rd 10.9.0.4:1
      route-target import evpn 1:10001
      route-target export evpn 1:10001
      redistribute connected
   !
   vrf GREEN
      rd 10.9.0.4:3
      route-target import evpn 3:10003
      route-target export evpn 3:10003
      redistribute connected
   !
   vrf RED
      rd 10.9.0.4:2
      route-target import evpn 2:10002
      route-target export evpn 2:10002
      redistribute connected
!
end
```
</details>

[Полная конфигурация DC2-Leaf-2-2](https://github.com/egorvshch/DC-networks-design/blob/main/project/configs/DC2-Leaf-2-2)
</details>

<details>
<summary> 4.7 Настройка DC2-Border-GW1 </summary>
-

<details>
<summary> Базовые настройки и настройки интерфейсов на DC2-Border-GW1 </summary>

```  
service routing protocols model multi-agent
!
hostname DC2-Border-GW1
!
vlan 10
   name service-10
!
vrf instance BLUE
vrf instance GREEN
vrf instance RED
!
interface Ethernet1
   description to DC2-Spine-1
   no switchport
   ip address 10.10.1.9/31
!
interface Ethernet2
   description to DC2-Spine-2
   no switchport
   ip address 10.10.2.9/31
!
interface Ethernet3
   description to DC2-Border-R1
   no switchport
!
interface Ethernet3.104
   encapsulation dot1q vlan 104
   vrf BLUE
   ip address 10.198.104.5/30
!
interface Ethernet3.105
   encapsulation dot1q vlan 105
   vrf RED
   ip address 10.198.105.5/30
!
interface Ethernet3.106
   encapsulation dot1q vlan 106
   vrf GREEN
   ip address 10.198.106.5/30
!
interface Ethernet4
   description to_DCI-Router-1
   no switchport
   ip address 10.2.91.5/31
!
interface Ethernet5
   description to_DCI-Router-2
   no switchport
   ip address 10.2.92.5/31
!
interface Ethernet6
   description to DC1-Border-R1
   no switchport
!
interface Ethernet6.114
   encapsulation dot1q vlan 114
   vrf BLUE
   ip address 10.198.104.13/30
!
interface Ethernet6.115
   encapsulation dot1q vlan 115
   vrf RED
   ip address 10.198.105.13/30
!
interface Ethernet6.116
   encapsulation dot1q vlan 116
   vrf GREEN
   ip address 10.198.106.13/30
!
interface Loopback0
   ip address 10.8.0.5/32
!
interface Loopback1
   ip address 10.9.0.5/32
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan virtual-router encapsulation mac-address 00:00:00:10:10:22
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vrf BLUE vni 11001
   vxlan vrf GREEN vni 11003
   vxlan vrf RED vni 11002
   vxlan learn-restrict any
!
ip virtual-router mac-address 00:00:00:00:00:01
!
   
```
</details>

<details>
<summary> Настройки маршрутизации BGP, nderlay и Overlay на DC2-Border-GW1 </summary>

```  

ip routing
ip routing vrf BLUE
ip routing vrf GREEN
ip routing vrf RED
!
ip prefix-list DIRECT_CONNECTED
   seq 10 permit 10.8.0.5/32
   seq 20 permit 10.9.0.5/32
!
route-map LOCAL_DIRECT_CONNECTED permit 10
   match ip address prefix-list DIRECT_CONNECTED
   match source-protocol connected
!
route-map RM_REDISTRIBUTE_CONNECTED permit 10
   match ip address prefix-list DIRECT_CONNECTED
!
router bgp 65298
   router-id 10.8.0.5
   timers bgp 3 9
   maximum-paths 4 ecmp 4
   neighbor DCI-OVERLAY peer group
   neighbor DCI-OVERLAY remote-as 65999
   neighbor DCI-OVERLAY update-source Loopback0
   neighbor DCI-OVERLAY ebgp-multihop 3
   neighbor DCI-OVERLAY send-community extended
   neighbor DCI-UNDERLAY peer group
   neighbor DCI-UNDERLAY remote-as 65999
   neighbor DCI-UNDERLAY route-map LOCAL_DIRECT_CONNECTED out
   neighbor DCI-UNDERLAY password 7 gyEXkFgefQGXVmT5Pf3u4Q==
   neighbor DCI-UNDERLAY send-community
   neighbor OVERLAY peer group
   neighbor OVERLAY remote-as 65200
   neighbor OVERLAY update-source Loopback0
   neighbor OVERLAY ebgp-multihop 3
   neighbor OVERLAY send-community extended
   neighbor UNDERLAY peer group
   neighbor UNDERLAY remote-as 65200
   neighbor UNDERLAY route-map LOCAL_DIRECT_CONNECTED out
   neighbor UNDERLAY password 7 4vMwMsSftvmv7pRrLIgVxA==
   neighbor UNDERLAY send-community
   neighbor 10.2.91.4 peer group DCI-UNDERLAY
   neighbor 10.2.92.4 peer group DCI-UNDERLAY
   neighbor 10.8.1.0 peer group OVERLAY
   neighbor 10.8.2.0 peer group OVERLAY
   neighbor 10.10.1.8 peer group UNDERLAY
   neighbor 10.10.2.8 peer group UNDERLAY
   neighbor 10.91.91.1 peer group DCI-OVERLAY
   neighbor 10.91.91.2 peer group DCI-OVERLAY
   redistribute connected route-map RM_REDISTRIBUTE_CONNECTED
   !
   vlan 10
      rd evpn domain all 10.8.0.5:20010
      route-target both 10:10010
      route-target import export evpn domain all 99999:10
      redistribute learned
   !
   address-family evpn
      neighbor DCI-OVERLAY activate
      neighbor DCI-OVERLAY domain remote
      neighbor OVERLAY activate
      neighbor default next-hop-self received-evpn-routes route-type ip-prefix inter-domain
   !
   address-family ipv4
      no neighbor DCI-OVERLAY activate
      no neighbor OVERLAY activate
   !
   vrf BLUE
      rd 10.9.0.5:1
      route-target import evpn 1:10001
      route-target export evpn 1:10001
      neighbor 10.198.104.6 remote-as 65301
      neighbor 10.198.104.6 password 7 SPai0bbFVUFsBkkZeLc/Hg==
      neighbor 10.198.104.14 remote-as 65301
      neighbor 10.198.104.14 password 7 n8PYVzGBFNJUEZN+cNG0Lg==
      aggregate-address 10.4.0.0/16 summary-only advertise-only
      aggregate-address 10.14.0.0/16 summary-only advertise-only
   !
   vrf GREEN
      rd 10.9.0.5:3
      route-target import evpn 3:10003
      route-target export evpn 3:10003
      neighbor 10.198.106.6 remote-as 65301
      neighbor 10.198.106.6 password 7 WfkUI+cquXJAi4nRIf/XlQ==
      neighbor 10.198.106.14 remote-as 65301
      neighbor 10.198.106.14 password 7 WhqEfFwjtFmU8fixDC13Fw==
      aggregate-address 10.6.0.0/16 summary-only advertise-only
      aggregate-address 10.13.0.0/16 summary-only advertise-only
   !
   vrf RED
      rd 10.9.0.5:2
      route-target import evpn 2:10002
      route-target export evpn 2:10002
      neighbor 10.198.105.6 remote-as 65301
      neighbor 10.198.105.6 password 7 SPai0bbFVUFsBkkZeLc/Hg==
      neighbor 10.198.105.14 remote-as 65301
      neighbor 10.198.105.14 password 7 n8PYVzGBFNJUEZN+cNG0Lg==
      aggregate-address 10.5.0.0/16 summary-only advertise-only
      aggregate-address 10.12.0.0/16 summary-only advertise-only
!
end
```
</details>

[Полная конфигурация DC2-Border-GW1](https://github.com/egorvshch/DC-networks-design/blob/main/project/configs/DC2-Border-GW1)
</details>

<details>
<summary> 4.8 Настройка DC2-Border-GW2 </summary>
-

<details>
<summary> Базовые настройки и настройки интерфейсов на DC2-Border-GW2 </summary>

```  
service routing protocols model multi-agent
!
hostname DC2-Border-GW2
!
vlan 10
   name service-10
!
vrf instance BLUE
vrf instance GREEN
vrf instance RED
!
interface Ethernet1
   description to DC2-Spine-1
   no switchport
   ip address 10.10.1.11/31
!
interface Ethernet2
   description to DC2-Spine-2
   no switchport
   ip address 10.10.2.11/31
!
interface Ethernet3
   description to DC2-Border-R1
   no switchport
!
interface Ethernet3.104
   encapsulation dot1q vlan 104
   vrf BLUE
   ip address 10.199.104.5/30
!
interface Ethernet3.105
   encapsulation dot1q vlan 105
   vrf RED
   ip address 10.199.105.5/30
!
interface Ethernet3.106
   encapsulation dot1q vlan 106
   vrf GREEN
   ip address 10.199.106.5/30
!
interface Ethernet4
   description to_DCI-Router-1
   no switchport
   ip address 10.2.91.7/31
!
interface Ethernet5
   description to_DCI-Router-2
   no switchport
   ip address 10.2.92.7/31
!
interface Ethernet6
   description to DC1-Border-R1
   no switchport
!
interface Ethernet6.114
   encapsulation dot1q vlan 114
   vrf BLUE
   ip address 10.199.104.13/30
!
interface Ethernet6.115
   encapsulation dot1q vlan 115
   vrf RED
   ip address 10.199.105.13/30
!
interface Ethernet6.116
   encapsulation dot1q vlan 116
   vrf GREEN
   ip address 10.199.106.13/30
!
interface Loopback0
   ip address 10.8.0.6/32
!
interface Loopback1
   ip address 10.9.0.5/32
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan virtual-router encapsulation mac-address 00:00:00:10:10:22
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vrf BLUE vni 11001
   vxlan vrf GREEN vni 11003
   vxlan vrf RED vni 11002
   vxlan learn-restrict any
!
ip virtual-router mac-address 00:00:00:00:00:01
!
   
```
</details>

<details>
<summary> Настройки маршрутизации BGP, nderlay и Overlay на DC2-Border-GW2 </summary>

```  

ip routing
ip routing vrf BLUE
ip routing vrf GREEN
ip routing vrf RED
!
ip prefix-list DIRECT_CONNECTED
   seq 10 permit 10.8.0.6/32
   seq 20 permit 10.9.0.6/32
   seq 30 permit 10.9.0.5/32
!
route-map LOCAL_DIRECT_CONNECTED permit 10
   match ip address prefix-list DIRECT_CONNECTED
   match source-protocol connected
!
route-map RM_REDISTRIBUTE_CONNECTED permit 10
   match ip address prefix-list DIRECT_CONNECTED
!
router bgp 65299
   router-id 10.8.0.6
   timers bgp 3 9
   maximum-paths 4 ecmp 4
   neighbor DCI-OVERLAY peer group
   neighbor DCI-OVERLAY remote-as 65999
   neighbor DCI-OVERLAY next-hop-unchanged
   neighbor DCI-OVERLAY update-source Loopback0
   neighbor DCI-OVERLAY ebgp-multihop 3
   neighbor DCI-OVERLAY send-community extended
   neighbor DCI-UNDERLAY peer group
   neighbor DCI-UNDERLAY remote-as 65999
   neighbor DCI-UNDERLAY route-map LOCAL_DIRECT_CONNECTED out
   neighbor DCI-UNDERLAY password 7 gyEXkFgefQGXVmT5Pf3u4Q==
   neighbor DCI-UNDERLAY send-community
   neighbor OVERLAY peer group
   neighbor OVERLAY remote-as 65200
   neighbor OVERLAY update-source Loopback0
   neighbor OVERLAY ebgp-multihop 3
   neighbor OVERLAY send-community extended
   neighbor UNDERLAY peer group
   neighbor UNDERLAY remote-as 65200
   neighbor UNDERLAY route-map LOCAL_DIRECT_CONNECTED out
   neighbor UNDERLAY password 7 4vMwMsSftvmv7pRrLIgVxA==
   neighbor UNDERLAY send-community
   neighbor 10.2.91.6 peer group DCI-UNDERLAY
   neighbor 10.2.92.6 peer group DCI-UNDERLAY
   neighbor 10.8.1.0 peer group OVERLAY
   neighbor 10.8.2.0 peer group OVERLAY
   neighbor 10.10.1.10 peer group UNDERLAY
   neighbor 10.10.2.10 peer group UNDERLAY
   neighbor 10.91.91.1 peer group DCI-OVERLAY
   neighbor 10.91.91.2 peer group DCI-OVERLAY
   redistribute connected route-map RM_REDISTRIBUTE_CONNECTED
   !
   vlan 10
      rd evpn domain all 10.8.0.6:20010
      route-target both 10:10010
      route-target import export evpn domain all 99999:10
      redistribute learned
   !
   address-family evpn
      neighbor DCI-OVERLAY activate
      neighbor DCI-OVERLAY domain remote
      neighbor OVERLAY activate
      neighbor default next-hop-self received-evpn-routes route-type ip-prefix inter-domain
   !
   address-family ipv4
      no neighbor DCI-OVERLAY activate
      no neighbor OVERLAY activate
   !
   vrf BLUE
      rd 10.9.0.6:1
      route-target import evpn 1:10001
      route-target export evpn 1:10001
      neighbor 10.199.104.6 remote-as 65301
      neighbor 10.199.104.6 password 7 SPai0bbFVUFsBkkZeLc/Hg==
      neighbor 10.199.104.14 remote-as 65301
      neighbor 10.199.104.14 password 7 n8PYVzGBFNJUEZN+cNG0Lg==
      aggregate-address 10.4.0.0/16 summary-only advertise-only
      aggregate-address 10.14.0.0/16 summary-only advertise-only
   !
   vrf GREEN
      rd 10.9.0.6:3
      route-target import evpn 3:10003
      route-target export evpn 3:10003
      neighbor 10.199.106.6 remote-as 65301
      neighbor 10.199.106.6 password 7 WfkUI+cquXJAi4nRIf/XlQ==
      neighbor 10.199.106.14 remote-as 65301
      neighbor 10.199.106.14 password 7 WhqEfFwjtFmU8fixDC13Fw==
      aggregate-address 10.6.0.0/16 summary-only advertise-only
      aggregate-address 10.13.0.0/16 summary-only advertise-only
   !
   vrf RED
      rd 10.9.0.6:2
      route-target import evpn 2:10002
      route-target export evpn 2:10002
      neighbor 10.199.105.6 remote-as 65301
      neighbor 10.199.105.6 password 7 SPai0bbFVUFsBkkZeLc/Hg==
      neighbor 10.199.105.14 remote-as 65301
      neighbor 10.199.105.14 password 7 n8PYVzGBFNJUEZN+cNG0Lg==
      aggregate-address 10.5.0.0/16 summary-only advertise-only
      aggregate-address 10.12.0.0/16 summary-only advertise-only
!
end
```
</details>

[Полная конфигурация DC2-Border-GW2](https://github.com/egorvshch/DC-networks-design/blob/main/project/configs/DC2-Border-GW2)
</details>

<details>
<summary> 4.9 Настройка DC2-Border-R1 </summary>
-

<details>
<summary> Базовые настройки и настройки интерфейсов на DC2-Border-R1 </summary>

```  
service routing protocols model multi-agent
!
hostname DC2-Border-R1
!
interface Ethernet1
   description to_DC2-Border-GW1
   no switchport
!
interface Ethernet1.104
   description to_vrf_BLUE_for_DC2-Border-GW1
   encapsulation dot1q vlan 104
   ip address 10.198.104.6/30
!
interface Ethernet1.105
   description to_vrf_RED_for_DC2-Border-GW1
   encapsulation dot1q vlan 105
   ip address 10.198.105.6/30
!
interface Ethernet1.106
   description to_vrf_GREEN_for_DC2-Border-GW1
   encapsulation dot1q vlan 106
   ip address 10.198.106.6/30
!
interface Ethernet2
   description to_DC2-Border-GW2
   no switchport
!
interface Ethernet2.104
   description to_vrf_BLUE_for_DC2-Border-GW2
   encapsulation dot1q vlan 104
   ip address 10.199.104.6/30
!
interface Ethernet2.105
   description to_vrf_RED_for_DC2-Border-GW2
   encapsulation dot1q vlan 105
   ip address 10.199.105.6/30
!
interface Ethernet2.106
   description to_vrf_GREEN_for_DC2-Border-GW2
   encapsulation dot1q vlan 106
   ip address 10.199.106.6/30
!
interface Ethernet3
   description to_DC1-Border-GW1
   no switchport
!
interface Ethernet3.114
   description to_vrf_BLUE_for_DC1-Border-GW1
   encapsulation dot1q vlan 114
   ip address 10.198.104.10/30
!
interface Ethernet3.115
   description to_vrf_RED_for_DC1-Border-GW1
   encapsulation dot1q vlan 115
   ip address 10.198.105.10/30
!
interface Ethernet3.116
   description to_vrf_GREEN_for_DC1-Border-GW1
   encapsulation dot1q vlan 116
   ip address 10.198.106.10/30
!
interface Ethernet4
   description to_DC1-Border-GW2
   no switchport
!
interface Ethernet4.114
   description to_vrf_BLUE_for_DC1-Border-GW2
   encapsulation dot1q vlan 114
   ip address 10.199.104.10/30
!
interface Ethernet4.115
   description to_vrf_RED_for_DC1-Border-GW2
   encapsulation dot1q vlan 115
   ip address 10.199.105.10/30
!
interface Ethernet4.116
   description to_vrf_GREEN_for_DC1-Border-GW2
   encapsulation dot1q vlan 116
   ip address 10.199.106.10/30
!
interface Ethernet5
   description to DC1-Border-R1
   no switchport
   ip address 10.195.0.2/30
!
interface Loopback0
   ip address 10.199.0.2/32
!
interface Loopback21
   ip address 2.2.2.2/32

!

```
</details>
<details>
<summary> Настройки маршрутизации BGP, Underlay и Overlay на DC2-Border-R1 </summary>

```  

ip routing
!
ip prefix-list LOOPBACKs
   seq 10 permit 2.2.2.2/32
!
ip prefix-list for_DC2-Border-GW
   seq 10 permit 0.0.0.0/0
!
route-map RM-REDISTRIB-CONNECTED permit 10
   match ip address prefix-list LOOPBACKs
!
route-map for_DC2-Border-GW_out permit 10
   match ip address prefix-list for_DC2-Border-GW
!
router bgp 65301
   bgp asn notation asdot
   router-id 10.199.0.2
   timers bgp 3 9
   maximum-paths 24
   neighbor DC1-Border-GW1 peer group
   neighbor DC1-Border-GW1 remote-as 65198
   neighbor DC1-Border-GW1 route-map for_DC2-Border-GW_out out
   neighbor DC1-Border-GW1 password 7 mXpeujoAlay5K6JXiQjWDQ==
   neighbor DC1-Border-GW1 default-originate
   neighbor DC1-Border-GW2 peer group
   neighbor DC1-Border-GW2 remote-as 65199
   neighbor DC1-Border-GW2 route-map for_DC2-Border-GW_out out
   neighbor DC1-Border-GW2 password 7 mvBH1FR+M+6kO9JylsFXRg==
   neighbor DC1-Border-GW2 default-originate
   neighbor DC2-Border-GW1 peer group
   neighbor DC2-Border-GW1 remote-as 65298
   neighbor DC2-Border-GW1 route-map for_DC2-Border-GW_out out
   neighbor DC2-Border-GW1 password 7 Jd5ObGtFDc7qf/cuFnrqTA==
   neighbor DC2-Border-GW1 default-originate
   neighbor DC2-Border-GW2 peer group
   neighbor DC2-Border-GW2 remote-as 65299
   neighbor DC2-Border-GW2 route-map for_DC2-Border-GW_out out
   neighbor DC2-Border-GW2 password 7 b9rsJJT1YnrVQNUUBHZtKw==
   neighbor DC2-Border-GW2 default-originate
   neighbor 10.195.0.1 remote-as 65301
   neighbor 10.195.0.1 next-hop-self
   neighbor 10.195.0.1 password 7 LU+TD0wE5HKh9/td5kqlJw==
   neighbor 10.198.104.5 peer group DC2-Border-GW1
   neighbor 10.198.104.9 peer group DC1-Border-GW1
   neighbor 10.198.105.5 peer group DC2-Border-GW1
   neighbor 10.198.105.9 peer group DC1-Border-GW1
   neighbor 10.198.106.5 peer group DC2-Border-GW1
   neighbor 10.198.106.9 peer group DC1-Border-GW1
   neighbor 10.199.104.5 peer group DC2-Border-GW2
   neighbor 10.199.104.9 peer group DC1-Border-GW2
   neighbor 10.199.105.5 peer group DC2-Border-GW2
   neighbor 10.199.105.9 peer group DC1-Border-GW2
   neighbor 10.199.106.5 peer group DC2-Border-GW2
   neighbor 10.199.106.9 peer group DC1-Border-GW2
   redistribute connected route-map RM-REDISTRIB-CONNECTED
!
end
```
</details>

[Полная конфигурация DC2-Border-R1](https://github.com/egorvshch/DC-networks-design/blob/main/project/configs/DC2-Border-R1)
</details>

<details>
<summary> 4.10 Настройка DC2-Host-1, DC2-Host-2, DC2-Host-3 и DC2-Host-4 </summary>

конфигурация хостов указана по сылками ниже:

[Полная конфигурация DC2-Host-1](https://github.com/egorvshch/DC-networks-design/blob/main/project/configs/DC2-Host-1)

[Полная конфигурация DC2-Host-2](https://github.com/egorvshch/DC-networks-design/blob/main/project/configs/DC2-Host-2)

[Полная конфигурация DC2-Host-3](https://github.com/egorvshch/DC-networks-design/blob/main/project/configs/DC2-Host-3)

[Полная конфигурация DC2-Host-4](https://github.com/egorvshch/DC-networks-design/blob/main/project/configs/DC2-Host-4)
</details>

### 5 Настройка обрудования DCI
<details>
<summary> 5.1 Настройка DCI-Router-1 </summary>
-
<details>
<summary> Базовые настройки и настройки интерфейсов на DCI-Router-1 </summary>

```  

service routing protocols model multi-agent
!
hostname DCI-Router-1
!
interface Ethernet1
   description to_DC1-Border-GW1
   no switchport
   ip address 10.2.91.0/31
!
interface Ethernet2
   description to_DC1-Border-GW2
   no switchport
   ip address 10.2.91.2/31
!
interface Ethernet3
   description to_DC2-Border-GW1
   no switchport
   ip address 10.2.91.4/31
!
interface Ethernet4
   description to_DC2-Border-GW2
   no switchport
   ip address 10.2.91.6/31
!
interface Loopback0
   ip address 10.91.91.1/32
!

```
</details>
<details>
<summary> Настройки маршрутизации BGP, Underlay и Overlay на DCI-Router-1 </summary>

```  
ip routing
!
ip prefix-list DIRECT_CONNECTED
   seq 10 permit 10.91.91.1/32
!
route-map RM_REDISTRIBUTE_CONNECTED permit 10
   match ip address prefix-list DIRECT_CONNECTED
!
router bgp 65999
   router-id 10.91.91.1
   timers bgp 3 9
   maximum-paths 4 ecmp 4
   neighbor DCI-OVERLAY peer group
   neighbor DCI-OVERLAY next-hop-unchanged
   neighbor DCI-OVERLAY update-source Loopback0
   neighbor DCI-OVERLAY ebgp-multihop 2
   neighbor DCI-OVERLAY send-community extended
   neighbor DCI-UNDERLAY peer group
   neighbor DCI-UNDERLAY password 7 gyEXkFgefQGXVmT5Pf3u4Q==
   neighbor DCI-UNDERLAY send-community
   neighbor 10.0.0.5 peer group DCI-OVERLAY
   neighbor 10.0.0.5 remote-as 65198
   neighbor 10.0.0.5 description DC1-Border-GW1
   neighbor 10.0.0.6 peer group DCI-OVERLAY
   neighbor 10.0.0.6 remote-as 65199
   neighbor 10.0.0.6 description DC1-Border-GW2
   neighbor 10.2.91.1 peer group DCI-UNDERLAY
   neighbor 10.2.91.1 remote-as 65198
   neighbor 10.2.91.1 description DC1-Border-GW1
   neighbor 10.2.91.3 peer group DCI-UNDERLAY
   neighbor 10.2.91.3 remote-as 65199
   neighbor 10.2.91.3 description DC1-Border-GW2
   neighbor 10.2.91.5 peer group DCI-UNDERLAY
   neighbor 10.2.91.5 remote-as 65298
   neighbor 10.2.91.5 description DC2-Border-GW1
   neighbor 10.2.91.7 peer group DCI-UNDERLAY
   neighbor 10.2.91.7 remote-as 65299
   neighbor 10.2.91.7 description DC2-Border-GW2
   neighbor 10.8.0.5 peer group DCI-OVERLAY
   neighbor 10.8.0.5 remote-as 65298
   neighbor 10.8.0.5 description DC2-Border-GW1
   neighbor 10.8.0.6 peer group DCI-OVERLAY
   neighbor 10.8.0.6 remote-as 65299
   neighbor 10.8.0.6 description DC2-Border-GW2
   redistribute connected route-map RM_REDISTRIBUTE_CONNECTED
   !
   address-family evpn
      neighbor DCI-OVERLAY activate
   !
   address-family ipv4
      no neighbor DCI-OVERLAY activate
!
end
```
</details>

[Полная конфигурация DCI-Router-1](https://github.com/egorvshch/DC-networks-design/blob/main/project/configs/DCI-Router-1)
</details>

<details>
<summary> 5.2 Настройка DCI-Router-2 </summary>
-
<details>
<summary> Базовые настройки и настройки интерфейсов на DCI-Router-2 </summary>

```  
!
service routing protocols model multi-agent
!
hostname DCI-Router-2
!
interface Ethernet1
   description to_DC1-Border-GW1
   no switchport
   ip address 10.2.92.0/31
!
interface Ethernet2
   description to_DC1-Border-GW2
   no switchport
   ip address 10.2.92.2/31
!
interface Ethernet3
   description to_DC2-Border-GW1
   no switchport
   ip address 10.2.92.4/31
!
interface Ethernet4
   description to_DC2-Border-GW2
   no switchport
   ip address 10.2.92.6/31
!
interface Loopback0
   ip address 10.91.91.2/32
!

```
</details>
<details>
<summary> Настройки маршрутизации BGP, Underlay и Overlay на DCI-Router-2 </summary>

```  
!
ip routing
!
ip prefix-list DIRECT_CONNECTED
   seq 10 permit 10.91.91.2/32
!
route-map RM_REDISTRIBUTE_CONNECTED permit 10
   match ip address prefix-list DIRECT_CONNECTED
!
router bgp 65999
   router-id 10.91.91.2
   timers bgp 3 9
   maximum-paths 4 ecmp 4
   neighbor DCI-OVERLAY peer group
   neighbor DCI-OVERLAY next-hop-unchanged
   neighbor DCI-OVERLAY update-source Loopback0
   neighbor DCI-OVERLAY ebgp-multihop 2
   neighbor DCI-OVERLAY send-community extended
   neighbor DCI-UNDERLAY peer group
   neighbor DCI-UNDERLAY password 7 gyEXkFgefQGXVmT5Pf3u4Q==
   neighbor DCI-UNDERLAY send-community
   neighbor 10.0.0.5 peer group DCI-OVERLAY
   neighbor 10.0.0.5 remote-as 65198
   neighbor 10.0.0.5 description DC1-Border-GW1
   neighbor 10.0.0.6 peer group DCI-OVERLAY
   neighbor 10.0.0.6 remote-as 65199
   neighbor 10.0.0.6 description DC1-Border-GW2
   neighbor 10.2.92.1 peer group DCI-UNDERLAY
   neighbor 10.2.92.1 remote-as 65198
   neighbor 10.2.92.1 description DC1-Border-GW1
   neighbor 10.2.92.3 peer group DCI-UNDERLAY
   neighbor 10.2.92.3 remote-as 65199
   neighbor 10.2.92.3 description DC1-Border-GW2
   neighbor 10.2.92.5 peer group DCI-UNDERLAY
   neighbor 10.2.92.5 remote-as 65298
   neighbor 10.2.92.5 description DC2-Border-GW1
   neighbor 10.2.92.7 peer group DCI-UNDERLAY
   neighbor 10.2.92.7 remote-as 65299
   neighbor 10.2.92.7 description DC2-Border-GW2
   neighbor 10.8.0.5 peer group DCI-OVERLAY
   neighbor 10.8.0.5 remote-as 65298
   neighbor 10.8.0.5 description DC2-Border-GW1
   neighbor 10.8.0.6 peer group DCI-OVERLAY
   neighbor 10.8.0.6 remote-as 65299
   neighbor 10.8.0.6 description DC2-Border-GW2
   redistribute connected route-map RM_REDISTRIBUTE_CONNECTED
   !
   address-family evpn
      neighbor DCI-OVERLAY activate
   !
   address-family ipv4
      no neighbor DCI-OVERLAY activate
!
end
```
</details>

[Полная конфигурация DCI-Router-2](https://github.com/egorvshch/DC-networks-design/blob/main/project/configs/DCI-Router-2)
</details>

### 6.Проверка настроек

<details>
<summary> 6.1 Проверка связности c DC1-Host-1 </summary>

```  
DC1-Host-1#ping 10.4.10.21 repeat 10
Type escape sequence to abort.
Sending 10, 100-byte ICMP Echos to 10.4.10.21, timeout is 2 seconds:
!!!!!!!!!!
Success rate is 100 percent (10/10), round-trip min/avg/max = 218/315/378 ms
DC1-Host-1#
DC1-Host-1#ping 10.4.10.21 repeat 10
Type escape sequence to abort.
Sending 10, 100-byte ICMP Echos to 10.4.10.21, timeout is 2 seconds:
!!!!!!!!!!
Success rate is 100 percent (10/10), round-trip min/avg/max = 183/284/477 ms
DC1-Host-1#
DC1-Host-1#ping 10.13.30.22 repeat 10
Type escape sequence to abort.
Sending 10, 100-byte ICMP Echos to 10.13.30.22, timeout is 2 seconds:
!!!!!!!!!!
Success rate is 100 percent (10/10), round-trip min/avg/max = 220/271/342 ms
DC1-Host-1#
DC1-Host-1#ping 10.13.30.23 repeat 10
Type escape sequence to abort.
Sending 10, 100-byte ICMP Echos to 10.13.30.23, timeout is 2 seconds:
!!!!!!!!!!
Success rate is 100 percent (10/10), round-trip min/avg/max = 180/413/861 ms
DC1-Host-1#
DC1-Host-1#ping 10.12.20.24 repeat 10
Type escape sequence to abort.
Sending 10, 100-byte ICMP Echos to 10.12.20.24, timeout is 2 seconds:
!!!!!!!!!!
Success rate is 100 percent (10/10), round-trip min/avg/max = 187/288/366 ms
DC1-Host-1#
DC1-Host-1#ping 10.14.14.254 repeat 10
Type escape sequence to abort.
Sending 10, 100-byte ICMP Echos to 10.14.14.254, timeout is 2 seconds:
!!!!!!!!!!
Success rate is 100 percent (10/10), round-trip min/avg/max = 125/290/414 ms
DC1-Host-1#

```
</details>

<details>
<summary> 6.2 Проверка на примере DC1-Leaf-1-1 </summary>

```
DC1-Leaf-1-1#
DC1-Leaf-1-1#sh bgp summary
BGP summary information for VRF default
Router identifier 10.0.0.1, local AS number 65101
Neighbor              AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc
------------ ----------- ------------- ----------------------- -------------- ---------- ----------
10.0.1.0           65100 Established   L2VPN EVPN              Negotiated             32         32
10.0.2.0           65100 Active        L2VPN EVPN              Configured              0          0
10.2.1.0           65100 Established   IPv4 Unicast            Negotiated              5          5
10.2.2.0           65100 Active        IPv4 Unicast            Configured              0          0
10.199.251.2       65101 Established   IPv4 Unicast            Negotiated              7          7
DC1-Leaf-1-1#
DC1-Leaf-1-1#show ip bgp
BGP routing table information for VRF default
Router identifier 10.0.0.1, local AS number 65101
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.0.0.1/32            -                     -       -          -       0       i
 * >      10.0.0.2/32            10.199.251.2          0       -          100     0       i
 * >      10.0.0.5/32            10.2.1.0              0       -          100     0       65100 65198 i
 *        10.0.0.5/32            10.199.251.2          0       -          100     0       65100 65198 i
 * >      10.0.0.6/32            10.2.1.0              0       -          100     0       65100 65199 i
 *        10.0.0.6/32            10.199.251.2          0       -          100     0       65100 65199 i
 * >      10.0.1.0/32            10.2.1.0              0       -          100     0       65100 i
 *        10.0.1.0/32            10.199.251.2          0       -          100     0       65100 i
 * >      10.1.0.1/32            -                     -       -          -       0       i
 * >      10.1.0.2/32            10.199.251.2          0       -          100     0       i
 * >      10.1.0.5/32            10.2.1.0              0       -          100     0       65100 65198 i
 *        10.1.0.5/32            10.199.251.2          0       -          100     0       65100 65198 i
 * >      10.1.1.0/32            10.2.1.0              0       -          100     0       65100 i
 *        10.1.1.0/32            10.199.251.2          0       -          100     0       65100 i
DC1-Leaf-1-1#
DC1-Leaf-1-1#show bgp evpn
BGP routing table information for VRF default
Router identifier 10.0.0.1, local AS number 65101
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 65101:10010 mac-ip aabb.cc80.6000
                                 -                     -       -       0       i
 * >      RD: 65101:10010 mac-ip aabb.cc80.6000 10.4.10.11
                                 -                     -       -       0       i
 * >      RD: 10.0.0.5:20010 mac-ip aabb.cc81.6000
                                 10.1.0.5              -       100     0       65100 65198 65999 65298 65200 65201 i
 * >      RD: 10.0.0.6:20010 mac-ip aabb.cc81.6000
                                 10.1.0.5              -       100     0       65100 65199 65999 65298 65200 65201 i
 * >      RD: 10.0.0.5:20010 mac-ip aabb.cc81.6000 10.4.10.21
                                 10.1.0.5              -       100     0       65100 65198 65999 65298 65200 65201 i
 * >      RD: 10.0.0.6:20010 mac-ip aabb.cc81.6000 10.4.10.21
                                 10.1.0.5              -       100     0       65100 65199 65999 65298 65200 65201 i
 * >      RD: 10.0.0.5:20010 mac-ip aabb.cc81.7000
                                 10.1.0.5              -       100     0       65100 65198 65999 65298 65200 65202 i
 * >      RD: 10.0.0.6:20010 mac-ip aabb.cc81.7000
                                 10.1.0.5              -       100     0       65100 65199 65999 65298 65200 65202 i
 * >      RD: 65101:10010 imet 10.1.0.1
                                 -                     -       -       0       i
 * >      RD: 65101:10020 imet 10.1.0.1
                                 -                     -       -       0       i
 * >      RD: 10.0.0.5:20010 imet 10.1.0.5
                                 10.1.0.5              -       100     0       65100 65198 i
 * >      RD: 10.0.0.6:20010 imet 10.1.0.5
                                 10.1.0.5              -       100     0       65100 65199 i
 * >      RD: 10.1.0.5:1 ip-prefix 0.0.0.0/0
                                 10.1.0.5              -       100     0       65100 65198 65301 ?
 * >      RD: 10.1.0.5:2 ip-prefix 0.0.0.0/0
                                 10.1.0.5              -       100     0       65100 65198 65301 ?
 * >      RD: 10.1.0.6:1 ip-prefix 0.0.0.0/0
                                 10.1.0.5              -       100     0       65100 65199 65301 ?
 * >      RD: 10.1.0.6:2 ip-prefix 0.0.0.0/0
                                 10.1.0.5              -       100     0       65100 65199 65301 ?
 * >      RD: 10.9.0.5:1 ip-prefix 10.4.0.0/16
                                 10.1.0.5              -       100     0       65100 65198 65999 65298 65200 i
 * >      RD: 10.9.0.6:1 ip-prefix 10.4.0.0/16
                                 10.1.0.5              -       100     0       65100 65199 65999 65299 65200 65201 i
 * >      RD: 10.1.0.1:1 ip-prefix 10.4.10.0/24
                                 -                     -       -       0       i
 * >      RD: 10.9.0.1:1 ip-prefix 10.4.10.0/24
                                 10.1.0.5              -       100     0       65100 65198 65999 65298 65200 65201 i
 * >      RD: 10.9.0.2:1 ip-prefix 10.4.10.0/24
                                 10.1.0.5              -       100     0       65100 65198 65999 65298 65200 65201 i
 * >      RD: 10.9.0.3:1 ip-prefix 10.4.10.0/24
                                 10.1.0.5              -       100     0       65100 65198 65999 65298 65200 65202 i
 * >      RD: 10.9.0.4:1 ip-prefix 10.4.10.0/24
                                 10.1.0.5              -       100     0       65100 65198 65999 65298 65200 65202 i
 * >      RD: 10.1.0.1:2 ip-prefix 10.5.20.0/24
                                 -                     -       -       0       i
 * >      RD: 10.1.0.5:2 ip-prefix 10.12.0.0/16
                                 10.1.0.5              -       100     0       65100 65198 65999 65298 65200 65201 i
 * >      RD: 10.1.0.6:2 ip-prefix 10.12.0.0/16
                                 10.1.0.5              -       100     0       65100 65199 65999 65298 65200 65201 i
 * >      RD: 10.9.0.5:2 ip-prefix 10.12.0.0/16
                                 10.1.0.5              -       100     0       65100 65198 65999 65298 65200 65201 i
 * >      RD: 10.9.0.6:2 ip-prefix 10.12.0.0/16
                                 10.1.0.5              -       100     0       65100 65198 65999 65299 65200 65201 i
 * >      RD: 10.9.0.1:2 ip-prefix 10.12.20.0/24
                                 10.1.0.5              -       100     0       65100 65198 65999 65298 65200 65201 i
 * >      RD: 10.9.0.2:2 ip-prefix 10.12.20.0/24
                                 10.1.0.5              -       100     0       65100 65198 65999 65298 65200 65201 i
 * >      RD: 10.9.0.3:2 ip-prefix 10.12.20.0/24
                                 10.1.0.5              -       100     0       65100 65198 65999 65298 65200 65202 i
 * >      RD: 10.9.0.4:2 ip-prefix 10.12.20.0/24
                                 10.1.0.5              -       100     0       65100 65198 65999 65298 65200 65202 i
 * >      RD: 10.1.0.5:1 ip-prefix 10.14.0.0/16
                                 10.1.0.5              -       100     0       65100 65198 65999 65298 65200 65201 i
 * >      RD: 10.1.0.6:1 ip-prefix 10.14.0.0/16
                                 10.1.0.5              -       100     0       65100 65199 65999 65298 65200 65201 i
 * >      RD: 10.9.0.5:1 ip-prefix 10.14.0.0/16
                                 10.1.0.5              -       100     0       65100 65199 65999 65298 65200 65201 i
 * >      RD: 10.9.0.6:1 ip-prefix 10.14.0.0/16
                                 10.1.0.5              -       100     0       65100 65199 65999 65299 65200 65201 i
 * >      RD: 10.9.0.1:1 ip-prefix 10.14.14.0/24
                                 10.1.0.5              -       100     0       65100 65198 65999 65298 65200 65201 i
 * >      RD: 10.9.0.2:1 ip-prefix 10.14.14.0/24
                                 10.1.0.5              -       100     0       65100 65198 65999 65298 65200 65201 i
DC1-Leaf-1-1#
DC1-Leaf-1-1#
DC1-Leaf-1-1#show ip route vrf BLUE

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
 B E      0.0.0.0/0 [200/0] via VTEP 10.1.0.5 VNI 11001 router-mac 00:00:00:10:10:11 local-interface Vxlan1

 C        10.4.10.0/24 is directly connected, Vlan10
 B E      10.4.0.0/16 [200/0] via VTEP 10.1.0.5 VNI 11001 router-mac 00:00:00:10:10:11 local-interface Vxlan1
 B E      10.14.14.0/24 [200/0] via VTEP 10.1.0.5 VNI 11001 router-mac 00:00:00:10:10:11 local-interface Vxlan1
 B E      10.14.0.0/16 [200/0] via VTEP 10.1.0.5 VNI 11001 router-mac 00:00:00:10:10:11 local-interface Vxlan1

DC1-Leaf-1-1#
DC1-Leaf-1-1#show ip route vrf RED

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
 B E      0.0.0.0/0 [200/0] via VTEP 10.1.0.5 VNI 11002 router-mac 00:00:00:10:10:11 local-interface Vxlan1

 C        10.5.20.0/24 is directly connected, Vlan20
 B E      10.12.20.0/24 [200/0] via VTEP 10.1.0.5 VNI 11002 router-mac 00:00:00:10:10:11 local-interface Vxlan1
 B E      10.12.0.0/16 [200/0] via VTEP 10.1.0.5 VNI 11002 router-mac 00:00:00:10:10:11 local-interface Vxlan1

DC1-Leaf-1-1#
DC1-Leaf-1-1#show mac address-table
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
  10    5000.0088.fe27    STATIC      Po999
  10    aabb.cc80.6000    DYNAMIC     Po10       1       0:07:00 ago
  10    aabb.cc81.6000    DYNAMIC     Vx1        1       0:06:45 ago
  10    aabb.cc81.7000    DYNAMIC     Vx1        1       22:29:00 ago
  20    5000.0088.fe27    STATIC      Po999
4090    0000.0000.0001    STATIC      Cpu
4090    5000.0088.fe27    STATIC      Po999
4091    0000.0000.0001    STATIC      Cpu
4091    5000.0088.fe27    STATIC      Po999
4093    0000.0000.0001    STATIC      Cpu
4093    0000.0010.1011    DYNAMIC     Vx1        1       22:28:58 ago
4094    0000.0000.0001    STATIC      Cpu
4094    0000.0010.1011    DYNAMIC     Vx1        1       22:28:59 ago
Total Mac Addresses for this criterion: 13

          Multicast Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       ----        -----
Total Mac Addresses for this criterion: 0
DC1-Leaf-1-1#
DC1-Leaf-1-1#
DC1-Leaf-1-1#show vxlan address-table
          Vxlan Mac Address Table
----------------------------------------------------------------------

VLAN  Mac Address     Type      Prt  VTEP             Moves   Last Move
----  -----------     ----      ---  ----             -----   ---------
  10  aabb.cc81.6000  EVPN      Vx1  10.1.0.5         1       0:06:51 ago
  10  aabb.cc81.7000  EVPN      Vx1  10.1.0.5         1       22:29:06 ago
4093  0000.0010.1011  EVPN      Vx1  10.1.0.5         1       22:29:04 ago
4094  0000.0010.1011  EVPN      Vx1  10.1.0.5         1       22:29:05 ago
Total Remote Mac Addresses for this criterion: 4
DC1-Leaf-1-1#
DC1-Leaf-1-1#
DC1-Leaf-1-1#show vxlan address-table
          Vxlan Mac Address Table
----------------------------------------------------------------------

VLAN  Mac Address     Type      Prt  VTEP             Moves   Last Move
----  -----------     ----      ---  ----             -----   ---------
  10  aabb.cc81.6000  EVPN      Vx1  10.1.0.5         1       0:07:34 ago
  10  aabb.cc81.7000  EVPN      Vx1  10.1.0.5         1       22:29:49 ago
4093  0000.0010.1011  EVPN      Vx1  10.1.0.5         1       22:29:47 ago
4094  0000.0010.1011  EVPN      Vx1  10.1.0.5         1       22:29:48 ago
Total Remote Mac Addresses for this criterion: 4
DC1-Leaf-1-1#
DC1-Leaf-1-1#show vxlan ?
  address-table   MAC forwarding table
  config-sanity   VXLAN Config Sanity
  control-plane   See control planes active in specific VLANs
  controller      VXLAN control service
  counters        VXLAN Counters
  flood           VXLAN flooding behavior
  learn-restrict  VXLAN learning restrictions
  qos             Qos settings
  security        Security related configuration
  vni             VNI to VLAN mapping
  vtep            VXLAN Tunnel End Points

DC1-Leaf-1-1#show vxlan flood vtep
          VXLAN Flood VTEP Table
--------------------------------------------------------------------------------

VLANS                            Ip Address
-----------------------------   ------------------------------------------------
10                              10.1.0.5
DC1-Leaf-1-1#show vxlan vtep
Remote VTEPS for Vxlan1:

VTEP           Tunnel Type(s)
-------------- --------------
10.1.0.5       flood, unicast

Total number of remote VTEPS:  1

DC1-Leaf-1-1#show vxlan vni
VNI to VLAN Mapping for Vxlan1
VNI         VLAN       Source       Interface            802.1Q Tag
----------- ---------- ------------ -------------------- ----------
10010       10         static       Ethernet3            10
                                    Ethernet5            10
                                    Port-Channel10       10
                                    Vxlan1               10
10020       20         static       Ethernet4            20
                                    Ethernet6            20
                                    Port-Channel20       20
                                    Vxlan1               20

VNI to dynamic VLAN Mapping for Vxlan1
VNI         VLAN       VRF        Source
----------- ---------- ---------- ------------
11001       4094       BLUE       evpn
11002       4093       RED        evpn

DC1-Leaf-1-1#
DC1-Leaf-1-1#
DC1-Leaf-1-1#show mlag
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
Active-full                        :                   2

DC1-Leaf-1-1#
DC1-Leaf-1-1#
DC1-Leaf-1-1#
DC1-Leaf-1-1#show mlag interfaces 10
                                                                   local/remote
  mlag      desc                 state       local       remote          status
--------- -------------- ---------------- ----------- ------------ ------------
    10      to Host-1      active-full        Po10         Po10           up/up
DC1-Leaf-1-1#
DC1-Leaf-1-1#show mlag interfaces 20
                                                                   local/remote
  mlag      desc                 state       local       remote          status
--------- -------------- ---------------- ----------- ------------ ------------
    20      to Host-2      active-full        Po20         Po20           up/up
DC1-Leaf-1-1#
DC1-Leaf-1-1#
DC1-Leaf-1-1#sh lacp 10 peer
State: A = Active, P = Passive; S=ShortTimeout, L=LongTimeout;
       G = Aggregable, I = Individual; s+=InSync, s-=OutOfSync;
       C = Collecting, X = state machine expired,
       D = Distributing, d = default neighbor state
                 |                        Partner
 Port    Status  | Sys-id                    Port#   State     OperKey  PortPri
------ ----------|------------------------- ------- --------- --------- -------
Port Channel Port-Channel10*:
 Et5     Bundled | 8000,aa-bb-cc-80-60-00        1   ALGs+CD    0x000a    32768

* - Only local interfaces for MLAGs are displayed. Connect to the peer to
    see the state for peer interfaces.
DC1-Leaf-1-1#sh lacp 20 peer
State: A = Active, P = Passive; S=ShortTimeout, L=LongTimeout;
       G = Aggregable, I = Individual; s+=InSync, s-=OutOfSync;
       C = Collecting, X = state machine expired,
       D = Distributing, d = default neighbor state
                 |                        Partner
 Port    Status  | Sys-id                    Port#   State     OperKey  PortPri
------ ----------|------------------------- ------- --------- --------- -------
Port Channel Port-Channel20*:
 Et6     Bundled | 8000,aa-bb-cc-80-90-00        1   ALGs+CD    0x0014    32768

* - Only local interfaces for MLAGs are displayed. Connect to the peer to
    see the state for peer interfaces.
DC1-Leaf-1-1#
DC1-Leaf-1-1#show bgp evpn route-type ip-prefix 0.0.0.0/0
BGP routing table information for VRF default
Router identifier 10.0.0.1, local AS number 65101
BGP routing table entry for ip-prefix 0.0.0.0/0, Route Distinguisher: 10.1.0.5:1
 Paths: 1 available
  65100 65198 65301
    10.1.0.5 from 10.0.1.0 (10.0.1.0)
      Origin INCOMPLETE, metric -, localpref 100, weight 0, tag 0, valid, external, best
      Extended Community: Route-Target-AS:1:10001 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:00:00:00:10:10:11
      VNI: 11001
BGP routing table entry for ip-prefix 0.0.0.0/0, Route Distinguisher: 10.1.0.5:2
 Paths: 1 available
  65100 65198 65301
    10.1.0.5 from 10.0.1.0 (10.0.1.0)
      Origin INCOMPLETE, metric -, localpref 100, weight 0, tag 0, valid, external, best
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:00:00:00:10:10:11
      VNI: 11002
BGP routing table entry for ip-prefix 0.0.0.0/0, Route Distinguisher: 10.1.0.6:1
 Paths: 1 available
  65100 65199 65301
    10.1.0.5 from 10.0.1.0 (10.0.1.0)
      Origin INCOMPLETE, metric -, localpref 100, weight 0, tag 0, valid, external, best
      Extended Community: Route-Target-AS:1:10001 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:00:00:00:10:10:11
      VNI: 11001
BGP routing table entry for ip-prefix 0.0.0.0/0, Route Distinguisher: 10.1.0.6:2
 Paths: 1 available
  65100 65199 65301
    10.1.0.5 from 10.0.1.0 (10.0.1.0)
      Origin INCOMPLETE, metric -, localpref 100, weight 0, tag 0, valid, external, best
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:00:00:00:10:10:11
      VNI: 11002

```
</details>

<details>
<summary> 6.3 Проверка на примере DC2-Leaf-1-1 </summary>

```
DC2-Leaf-1-1#
DC2-Leaf-1-1#
DC2-Leaf-1-1#sh bgp summary
BGP summary information for VRF default
Router identifier 10.8.0.1, local AS number 65201
Neighbor              AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc
------------ ----------- ------------- ----------------------- -------------- ---------- ----------
10.8.1.0           65200 Established   L2VPN EVPN              Negotiated             43         43
10.8.2.0           65200 Established   L2VPN EVPN              Negotiated             43         43
10.10.1.0          65200 Established   IPv4 Unicast            Negotiated              9          9
10.10.2.0          65200 Established   IPv4 Unicast            Negotiated              9          9
10.199.253.2       65201 Established   IPv4 Unicast            Negotiated             13         13
DC2-Leaf-1-1#
DC2-Leaf-1-1#show ip bgp
BGP routing table information for VRF default
Router identifier 10.8.0.1, local AS number 65201
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.8.0.1/32            -                     -       -          -       0       i
 * >      10.8.0.2/32            10.199.253.2          0       -          100     0       i
 * >Ec    10.8.0.3/32            10.10.1.0             0       -          100     0       65200 65202 i
 *  ec    10.8.0.3/32            10.10.2.0             0       -          100     0       65200 65202 i
 *        10.8.0.3/32            10.199.253.2          0       -          100     0       65200 65202 i
 * >Ec    10.8.0.4/32            10.10.1.0             0       -          100     0       65200 65202 i
 *  ec    10.8.0.4/32            10.10.2.0             0       -          100     0       65200 65202 i
 *        10.8.0.4/32            10.199.253.2          0       -          100     0       65200 65202 i
 * >Ec    10.8.0.5/32            10.10.1.0             0       -          100     0       65200 65298 i
 *  ec    10.8.0.5/32            10.10.2.0             0       -          100     0       65200 65298 i
 *        10.8.0.5/32            10.199.253.2          0       -          100     0       65200 65298 i
 * >Ec    10.8.0.6/32            10.10.1.0             0       -          100     0       65200 65299 i
 *  ec    10.8.0.6/32            10.10.2.0             0       -          100     0       65200 65299 i
 *        10.8.0.6/32            10.199.253.2          0       -          100     0       65200 65299 i
 * >      10.8.1.0/32            10.10.1.0             0       -          100     0       65200 i
 *        10.8.1.0/32            10.199.253.2          0       -          100     0       65200 i
 * >      10.8.2.0/32            10.10.2.0             0       -          100     0       65200 i
 *        10.8.2.0/32            10.199.253.2          0       -          100     0       65200 i
 * >      10.9.0.1/32            -                     -       -          -       0       i
 * >      10.9.0.2/32            10.199.253.2          0       -          100     0       i
 * >Ec    10.9.0.3/32            10.10.1.0             0       -          100     0       65200 65202 i
 *  ec    10.9.0.3/32            10.10.2.0             0       -          100     0       65200 65202 i
 *        10.9.0.3/32            10.199.253.2          0       -          100     0       65200 65202 i
 * >Ec    10.9.0.4/32            10.10.1.0             0       -          100     0       65200 65202 i
 *  ec    10.9.0.4/32            10.10.2.0             0       -          100     0       65200 65202 i
 *        10.9.0.4/32            10.199.253.2          0       -          100     0       65200 65202 i
 * >Ec    10.9.0.5/32            10.10.1.0             0       -          100     0       65200 65298 i
 *  ec    10.9.0.5/32            10.10.2.0             0       -          100     0       65200 65298 i
 *        10.9.0.5/32            10.199.253.2          0       -          100     0       65200 65298 i
 * >      10.9.1.0/32            10.10.1.0             0       -          100     0       65200 i
 *        10.9.1.0/32            10.199.253.2          0       -          100     0       65200 i
 * >      10.9.2.0/32            10.10.2.0             0       -          100     0       65200 i
 *        10.9.2.0/32            10.199.253.2          0       -          100     0       65200 i
DC2-Leaf-1-1#
DC2-Leaf-1-1#show bgp evpn
BGP routing table information for VRF default
Router identifier 10.8.0.1, local AS number 65201
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.8.0.5:20010 mac-ip aabb.cc80.6000
                                 10.9.0.5              -       100     0       65200 65298 65999 65198 65100 65101 i
 *  ec    RD: 10.8.0.5:20010 mac-ip aabb.cc80.6000
                                 10.9.0.5              -       100     0       65200 65298 65999 65198 65100 65101 i
 * >Ec    RD: 10.8.0.6:20010 mac-ip aabb.cc80.6000
                                 10.9.0.5              -       100     0       65200 65299 65999 65198 65100 65101 i
 *  ec    RD: 10.8.0.6:20010 mac-ip aabb.cc80.6000
                                 10.9.0.5              -       100     0       65200 65299 65999 65198 65100 65101 i
 * >Ec    RD: 10.8.0.5:20010 mac-ip aabb.cc80.6000 10.4.10.11
                                 10.9.0.5              -       100     0       65200 65298 65999 65199 65100 65101 i
 *  ec    RD: 10.8.0.5:20010 mac-ip aabb.cc80.6000 10.4.10.11
                                 10.9.0.5              -       100     0       65200 65298 65999 65199 65100 65101 i
 * >Ec    RD: 10.8.0.6:20010 mac-ip aabb.cc80.6000 10.4.10.11
                                 10.9.0.5              -       100     0       65200 65299 65999 65199 65100 65101 i
 *  ec    RD: 10.8.0.6:20010 mac-ip aabb.cc80.6000 10.4.10.11
                                 10.9.0.5              -       100     0       65200 65299 65999 65199 65100 65101 i
 * >      RD: 10.9.0.1:10010 mac-ip aabb.cc81.6000
                                 -                     -       -       0       i
 * >      RD: 10.9.0.1:10010 mac-ip aabb.cc81.6000 10.4.10.21
                                 -                     -       -       0       i
 * >Ec    RD: 10.9.0.3:10030 mac-ip aabb.cc81.7000
                                 10.9.0.3              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.3:10030 mac-ip aabb.cc81.7000
                                 10.9.0.3              -       100     0       65200 65202 i
 * >Ec    RD: 10.9.0.4:10010 mac-ip aabb.cc81.7000
                                 10.9.0.4              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.4:10010 mac-ip aabb.cc81.7000
                                 10.9.0.4              -       100     0       65200 65202 i
 * >Ec    RD: 10.9.0.4:10030 mac-ip aabb.cc81.7000
                                 10.9.0.4              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.4:10030 mac-ip aabb.cc81.7000
                                 10.9.0.4              -       100     0       65200 65202 i
 * >Ec    RD: 10.9.0.3:10030 mac-ip aabb.cc81.7000 10.13.30.23
                                 10.9.0.3              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.3:10030 mac-ip aabb.cc81.7000 10.13.30.23
                                 10.9.0.3              -       100     0       65200 65202 i
 * >Ec    RD: 10.9.0.4:10030 mac-ip aabb.cc81.7000 10.13.30.23
                                 10.9.0.4              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.4:10030 mac-ip aabb.cc81.7000 10.13.30.23
                                 10.9.0.4              -       100     0       65200 65202 i
 * >Ec    RD: 10.9.0.3:10020 mac-ip aabb.cc81.b000
                                 10.9.0.3              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.3:10020 mac-ip aabb.cc81.b000
                                 10.9.0.3              -       100     0       65200 65202 i
 * >Ec    RD: 10.9.0.4:10020 mac-ip aabb.cc81.b000
                                 10.9.0.4              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.4:10020 mac-ip aabb.cc81.b000
                                 10.9.0.4              -       100     0       65200 65202 i
 * >Ec    RD: 10.9.0.3:10020 mac-ip aabb.cc81.b000 10.12.20.24
                                 10.9.0.3              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.3:10020 mac-ip aabb.cc81.b000 10.12.20.24
                                 10.9.0.3              -       100     0       65200 65202 i
 * >Ec    RD: 10.9.0.4:10020 mac-ip aabb.cc81.b000 10.12.20.24
                                 10.9.0.4              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.4:10020 mac-ip aabb.cc81.b000 10.12.20.24
                                 10.9.0.4              -       100     0       65200 65202 i
 * >      RD: 10.9.0.1:10030 mac-ip aabb.cc81.c000
                                 -                     -       -       0       i
 * >      RD: 10.9.0.1:10030 mac-ip aabb.cc81.c000 10.13.30.22
                                 -                     -       -       0       i
 * >      RD: 10.9.0.1:10010 imet 10.9.0.1
                                 -                     -       -       0       i
 * >      RD: 10.9.0.1:10020 imet 10.9.0.1
                                 -                     -       -       0       i
 * >      RD: 10.9.0.1:10030 imet 10.9.0.1
                                 -                     -       -       0       i
 * >Ec    RD: 10.9.0.3:10010 imet 10.9.0.3
                                 10.9.0.3              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.3:10010 imet 10.9.0.3
                                 10.9.0.3              -       100     0       65200 65202 i
 * >Ec    RD: 10.9.0.3:10020 imet 10.9.0.3
                                 10.9.0.3              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.3:10020 imet 10.9.0.3
                                 10.9.0.3              -       100     0       65200 65202 i
 * >Ec    RD: 10.9.0.3:10030 imet 10.9.0.3
                                 10.9.0.3              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.3:10030 imet 10.9.0.3
                                 10.9.0.3              -       100     0       65200 65202 i
 * >Ec    RD: 10.9.0.4:10010 imet 10.9.0.4
                                 10.9.0.4              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.4:10010 imet 10.9.0.4
                                 10.9.0.4              -       100     0       65200 65202 i
 * >Ec    RD: 10.9.0.4:10020 imet 10.9.0.4
                                 10.9.0.4              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.4:10020 imet 10.9.0.4
                                 10.9.0.4              -       100     0       65200 65202 i
 * >Ec    RD: 10.9.0.4:10030 imet 10.9.0.4
                                 10.9.0.4              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.4:10030 imet 10.9.0.4
                                 10.9.0.4              -       100     0       65200 65202 i
 * >Ec    RD: 10.8.0.5:20010 imet 10.9.0.5
                                 10.9.0.5              -       100     0       65200 65298 i
 *  ec    RD: 10.8.0.5:20010 imet 10.9.0.5
                                 10.9.0.5              -       100     0       65200 65298 i
 * >Ec    RD: 10.8.0.6:20010 imet 10.9.0.5
                                 10.9.0.5              -       100     0       65200 65299 i
 *  ec    RD: 10.8.0.6:20010 imet 10.9.0.5
                                 10.9.0.5              -       100     0       65200 65299 i
 * >Ec    RD: 10.9.0.5:1 ip-prefix 0.0.0.0/0
                                 10.9.0.5              -       100     0       65200 65298 65301 ?
 *  ec    RD: 10.9.0.5:1 ip-prefix 0.0.0.0/0
                                 10.9.0.5              -       100     0       65200 65298 65301 ?
 * >Ec    RD: 10.9.0.5:2 ip-prefix 0.0.0.0/0
                                 10.9.0.5              -       100     0       65200 65298 65301 ?
 *  ec    RD: 10.9.0.5:2 ip-prefix 0.0.0.0/0
                                 10.9.0.5              -       100     0       65200 65298 65301 ?
 * >Ec    RD: 10.9.0.5:3 ip-prefix 0.0.0.0/0
                                 10.9.0.5              -       100     0       65200 65298 65301 ?
 *  ec    RD: 10.9.0.5:3 ip-prefix 0.0.0.0/0
                                 10.9.0.5              -       100     0       65200 65298 65301 ?
 * >Ec    RD: 10.9.0.6:1 ip-prefix 0.0.0.0/0
                                 10.9.0.5              -       100     0       65200 65299 65301 ?
 *  ec    RD: 10.9.0.6:1 ip-prefix 0.0.0.0/0
                                 10.9.0.5              -       100     0       65200 65299 65301 ?
 * >Ec    RD: 10.9.0.6:2 ip-prefix 0.0.0.0/0
                                 10.9.0.5              -       100     0       65200 65299 65301 ?
 *  ec    RD: 10.9.0.6:2 ip-prefix 0.0.0.0/0
                                 10.9.0.5              -       100     0       65200 65299 65301 ?
 * >Ec    RD: 10.9.0.6:3 ip-prefix 0.0.0.0/0
                                 10.9.0.5              -       100     0       65200 65299 65301 ?
 *  ec    RD: 10.9.0.6:3 ip-prefix 0.0.0.0/0
                                 10.9.0.5              -       100     0       65200 65299 65301 ?
 * >Ec    RD: 10.1.0.5:1 ip-prefix 10.4.0.0/16
                                 10.9.0.5              -       100     0       65200 65299 65999 65198 65100 65101 i
 *  ec    RD: 10.1.0.5:1 ip-prefix 10.4.0.0/16
                                 10.9.0.5              -       100     0       65200 65299 65999 65198 65100 65101 i
 * >Ec    RD: 10.1.0.6:1 ip-prefix 10.4.0.0/16
                                 10.9.0.5              -       100     0       65200 65299 65999 65199 65100 65101 i
 *  ec    RD: 10.1.0.6:1 ip-prefix 10.4.0.0/16
                                 10.9.0.5              -       100     0       65200 65299 65999 65199 65100 65101 i
 * >Ec    RD: 10.1.0.1:1 ip-prefix 10.4.10.0/24
                                 10.9.0.5              -       100     0       65200 65298 65999 65198 65100 65101 i
 *  ec    RD: 10.1.0.1:1 ip-prefix 10.4.10.0/24
                                 10.9.0.5              -       100     0       65200 65298 65999 65198 65100 65101 i
 * >Ec    RD: 10.1.0.2:1 ip-prefix 10.4.10.0/24
                                 10.9.0.5              -       100     0       65200 65298 65999 65198 65100 65101 i
 *  ec    RD: 10.1.0.2:1 ip-prefix 10.4.10.0/24
                                 10.9.0.5              -       100     0       65200 65298 65999 65198 65100 65101 i
 * >      RD: 10.9.0.1:1 ip-prefix 10.4.10.0/24
                                 -                     -       -       0       i
 * >Ec    RD: 10.9.0.3:1 ip-prefix 10.4.10.0/24
                                 10.9.0.3              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.3:1 ip-prefix 10.4.10.0/24
                                 10.9.0.3              -       100     0       65200 65202 i
 * >Ec    RD: 10.9.0.4:1 ip-prefix 10.4.10.0/24
                                 10.9.0.4              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.4:1 ip-prefix 10.4.10.0/24
                                 10.9.0.4              -       100     0       65200 65202 i
 * >Ec    RD: 10.1.0.5:2 ip-prefix 10.5.0.0/16
                                 10.9.0.5              -       100     0       65200 65299 65999 65198 65100 65101 i
 *  ec    RD: 10.1.0.5:2 ip-prefix 10.5.0.0/16
                                 10.9.0.5              -       100     0       65200 65299 65999 65198 65100 65101 i
 * >Ec    RD: 10.1.0.6:2 ip-prefix 10.5.0.0/16
                                 10.9.0.5              -       100     0       65200 65299 65999 65199 65100 65101 i
 *  ec    RD: 10.1.0.6:2 ip-prefix 10.5.0.0/16
                                 10.9.0.5              -       100     0       65200 65299 65999 65199 65100 65101 i
 * >Ec    RD: 10.9.0.5:2 ip-prefix 10.5.0.0/16
                                 10.9.0.5              -       100     0       65200 65298 65999 65198 65100 65101 i
 *  ec    RD: 10.9.0.5:2 ip-prefix 10.5.0.0/16
                                 10.9.0.5              -       100     0       65200 65298 65999 65198 65100 65101 i
 * >Ec    RD: 10.9.0.6:2 ip-prefix 10.5.0.0/16
                                 10.9.0.5              -       100     0       65200 65299 65999 65198 65100 65101 i
 *  ec    RD: 10.9.0.6:2 ip-prefix 10.5.0.0/16
                                 10.9.0.5              -       100     0       65200 65299 65999 65198 65100 65101 i
 * >Ec    RD: 10.1.0.1:2 ip-prefix 10.5.20.0/24
                                 10.9.0.5              -       100     0       65200 65298 65999 65198 65100 65101 i
 *  ec    RD: 10.1.0.1:2 ip-prefix 10.5.20.0/24
                                 10.9.0.5              -       100     0       65200 65298 65999 65198 65100 65101 i
 * >Ec    RD: 10.1.0.2:2 ip-prefix 10.5.20.0/24
                                 10.9.0.5              -       100     0       65200 65298 65999 65198 65100 65101 i
 *  ec    RD: 10.1.0.2:2 ip-prefix 10.5.20.0/24
                                 10.9.0.5              -       100     0       65200 65298 65999 65198 65100 65101 i
 * >      RD: 10.9.0.1:2 ip-prefix 10.12.20.0/24
                                 -                     -       -       0       i
 * >Ec    RD: 10.9.0.3:2 ip-prefix 10.12.20.0/24
                                 10.9.0.3              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.3:2 ip-prefix 10.12.20.0/24
                                 10.9.0.3              -       100     0       65200 65202 i
 * >Ec    RD: 10.9.0.4:2 ip-prefix 10.12.20.0/24
                                 10.9.0.4              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.4:2 ip-prefix 10.12.20.0/24
                                 10.9.0.4              -       100     0       65200 65202 i
 * >      RD: 10.9.0.1:3 ip-prefix 10.13.30.0/24
                                 -                     -       -       0       i
 * >Ec    RD: 10.9.0.3:3 ip-prefix 10.13.30.0/24
                                 10.9.0.3              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.3:3 ip-prefix 10.13.30.0/24
                                 10.9.0.3              -       100     0       65200 65202 i
 * >Ec    RD: 10.9.0.4:3 ip-prefix 10.13.30.0/24
                                 10.9.0.4              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.4:3 ip-prefix 10.13.30.0/24
                                 10.9.0.4              -       100     0       65200 65202 i
 * >      RD: 10.9.0.1:1 ip-prefix 10.14.14.0/24
                                 -                     -       -       0       i
DC2-Leaf-1-1#
DC2-Leaf-1-1#
DC2-Leaf-1-1#show ip route vrf BLUE

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
 B E      0.0.0.0/0 [200/0] via VTEP 10.9.0.5 VNI 11001 router-mac 00:00:00:10:10:22 local-interface Vxlan1

 C        10.4.10.0/24 is directly connected, Vlan10
 B E      10.4.0.0/16 [200/0] via VTEP 10.9.0.5 VNI 11001 router-mac 00:00:00:10:10:22 local-interface Vxlan1
 C        10.14.14.0/24 is directly connected, Vlan14

DC2-Leaf-1-1#show ip route vrf RED

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
 B E      0.0.0.0/0 [200/0] via VTEP 10.9.0.5 VNI 11002 router-mac 00:00:00:10:10:22 local-interface Vxlan1

 B E      10.5.20.0/24 [200/0] via VTEP 10.9.0.5 VNI 11002 router-mac 00:00:00:10:10:22 local-interface Vxlan1
 B E      10.5.0.0/16 [200/0] via VTEP 10.9.0.5 VNI 11002 router-mac 00:00:00:10:10:22 local-interface Vxlan1
 B E      10.12.20.24/32 [200/0] via VTEP 10.9.0.3 VNI 11002 router-mac 50:00:00:09:ef:21 local-interface Vxlan1
                                 via VTEP 10.9.0.4 VNI 11002 router-mac 50:00:00:d5:e2:ad local-interface Vxlan1
 C        10.12.20.0/24 is directly connected, Vlan20

DC2-Leaf-1-1#show ip route vrf GREEN

VRF: GREEN
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
 B E      0.0.0.0/0 [200/0] via VTEP 10.9.0.5 VNI 11003 router-mac 00:00:00:10:10:22 local-interface Vxlan1

 B E      10.13.30.23/32 [200/0] via VTEP 10.9.0.3 VNI 11003 router-mac 50:00:00:09:ef:21 local-interface Vxlan1
                                 via VTEP 10.9.0.4 VNI 11003 router-mac 50:00:00:d5:e2:ad local-interface Vxlan1
 C        10.13.30.0/24 is directly connected, Vlan30

DC2-Leaf-1-1#
DC2-Leaf-1-1#
DC2-Leaf-1-1#show mac address-table
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
  10    5000.0043.40cf    STATIC      Po999
  10    aabb.cc80.6000    DYNAMIC     Vx1        1       0:01:38 ago
  10    aabb.cc81.6000    DYNAMIC     Po10       1       0:00:53 ago
  10    aabb.cc81.7000    DYNAMIC     Vx1        1       1 day, 2:59:35 ago
  14    0000.0000.0001    STATIC      Cpu
  14    5000.0043.40cf    STATIC      Po999
  20    0000.0000.0001    STATIC      Cpu
  20    5000.0043.40cf    STATIC      Po999
  20    aabb.cc81.b000    DYNAMIC     Vx1        1       0:01:05 ago
  30    0000.0000.0001    STATIC      Cpu
  30    5000.0043.40cf    STATIC      Po999
  30    aabb.cc81.7000    DYNAMIC     Vx1        1       0:01:37 ago
  30    aabb.cc81.c000    DYNAMIC     Po30       1       0:01:37 ago
4090    0000.0000.0001    STATIC      Cpu
4091    0000.0000.0001    STATIC      Cpu
4092    0000.0000.0001    STATIC      Cpu
4092    0000.0010.1022    DYNAMIC     Vx1        1       20:40:05 ago
4092    5000.0009.ef21    DYNAMIC     Vx1        1       1 day, 3:01:09 ago
4092    5000.00d5.e2ad    DYNAMIC     Vx1        1       1 day, 2:59:35 ago
4093    0000.0000.0001    STATIC      Cpu
4093    0000.0010.1022    DYNAMIC     Vx1        1       22:41:40 ago
4093    5000.0009.ef21    DYNAMIC     Vx1        1       1 day, 3:01:09 ago
4093    5000.00d5.e2ad    DYNAMIC     Vx1        1       1 day, 2:59:36 ago
4094    0000.0000.0001    STATIC      Cpu
4094    0000.0010.1022    DYNAMIC     Vx1        1       22:41:42 ago
4094    5000.0009.ef21    DYNAMIC     Vx1        1       1 day, 3:01:08 ago
4094    5000.00d5.e2ad    DYNAMIC     Vx1        1       1 day, 2:59:35 ago
Total Mac Addresses for this criterion: 27

          Multicast Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       ----        -----
Total Mac Addresses for this criterion: 0
DC2-Leaf-1-1#show vxlan address-table
          Vxlan Mac Address Table
----------------------------------------------------------------------

VLAN  Mac Address     Type      Prt  VTEP             Moves   Last Move
----  -----------     ----      ---  ----             -----   ---------
  10  aabb.cc80.6000  EVPN      Vx1  10.9.0.5         1       0:01:46 ago
  10  aabb.cc81.7000  EVPN      Vx1  10.9.0.4         1       1 day, 2:59:42 ago
  20  aabb.cc81.b000  EVPN      Vx1  10.9.0.3         1       0:01:12 ago
  30  aabb.cc81.7000  EVPN      Vx1  10.9.0.3         1       0:01:44 ago
4092  0000.0010.1022  EVPN      Vx1  10.9.0.5         1       20:40:12 ago
4092  5000.0009.ef21  EVPN      Vx1  10.9.0.3         1       1 day, 3:01:16 ago
4092  5000.00d5.e2ad  EVPN      Vx1  10.9.0.4         1       1 day, 2:59:43 ago
4093  0000.0010.1022  EVPN      Vx1  10.9.0.5         1       22:41:47 ago
4093  5000.0009.ef21  EVPN      Vx1  10.9.0.3         1       1 day, 3:01:16 ago
4093  5000.00d5.e2ad  EVPN      Vx1  10.9.0.4         1       1 day, 2:59:44 ago
4094  0000.0010.1022  EVPN      Vx1  10.9.0.5         1       22:41:49 ago
4094  5000.0009.ef21  EVPN      Vx1  10.9.0.3         1       1 day, 3:01:16 ago
4094  5000.00d5.e2ad  EVPN      Vx1  10.9.0.4         1       1 day, 2:59:43 ago
Total Remote Mac Addresses for this criterion: 13
DC2-Leaf-1-1#show vxlan vtep
Remote VTEPS for Vxlan1:

VTEP           Tunnel Type(s)
-------------- --------------
10.9.0.3       flood, unicast
10.9.0.4       flood, unicast
10.9.0.5       flood, unicast

Total number of remote VTEPS:  3
DC2-Leaf-1-1#show vxlan flood vtep
          VXLAN Flood VTEP Table
--------------------------------------------------------------------------------

VLANS                            Ip Address
-----------------------------   ------------------------------------------------
10                              10.9.0.3        10.9.0.4        10.9.0.5
20,30                           10.9.0.3        10.9.0.4
DC2-Leaf-1-1#
DC2-Leaf-1-1#show vxlan vni
VNI to VLAN Mapping for Vxlan1
VNI         VLAN       Source       Interface            802.1Q Tag
----------- ---------- ------------ -------------------- ----------
10010       10         static       Ethernet3            10
                                    Ethernet5            10
                                    Port-Channel10       10
                                    Vxlan1               10
10020       20         static       Ethernet4            20
                                    Port-Channel20       20
                                    Vxlan1               20
10030       30         static       Ethernet6            30
                                    Port-Channel30       30
                                    Vxlan1               30

VNI to dynamic VLAN Mapping for Vxlan1
VNI         VLAN       VRF         Source
----------- ---------- ----------- ------------
11001       4094       BLUE        evpn
11002       4093       RED         evpn
11003       4092       GREEN       evpn

DC2-Leaf-1-1#
DC2-Leaf-1-1#
DC2-Leaf-1-1#show mlag
MLAG Configuration:
domain-id                          :            mlag-999
local-interface                    :            Vlan4090
peer-address                       :        10.199.252.2
peer-link                          :     Port-Channel999
peer-config                        :          consistent

MLAG Status:
state                              :              Active
negotiation status                 :           Connected
peer-link status                   :                  Up
local-int status                   :                  Up
system-id                          :   52:00:00:43:40:cf
dual-primary detection             :            Disabled
dual-primary interface errdisabled :               False

MLAG Ports:
Disabled                           :                   0
Configured                         :                   0
Inactive                           :                   1
Active-partial                     :                   0
Active-full                        :                   2

DC2-Leaf-1-1#
DC2-Leaf-1-1#
DC2-Leaf-1-1#show mlag interfaces 10
                                                                   local/remote
  mlag     desc                    state      local      remote          status
-------- ----------------- ---------------- ---------- ----------- ------------
    10     to DC2-Host-1     active-full       Po10        Po10           up/up
DC2-Leaf-1-1#show mlag interfaces 30
                                                                   local/remote
  mlag     desc                    state      local      remote          status
-------- ----------------- ---------------- ---------- ----------- ------------
    30     to DC2-Host-2     active-full       Po30        Po30           up/up
DC2-Leaf-1-1#

DC2-Leaf-1-1#show bgp evpn route-type ip-prefix 0.0.0.0/0
BGP routing table information for VRF default
Router identifier 10.8.0.1, local AS number 65201
BGP routing table entry for ip-prefix 0.0.0.0/0, Route Distinguisher: 10.9.0.5:1
 Paths: 2 available
  65200 65298 65301
    10.9.0.5 from 10.8.1.0 (10.8.1.0)
      Origin INCOMPLETE, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:1:10001 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:00:00:00:10:10:22
      VNI: 11001
  65200 65298 65301
    10.9.0.5 from 10.8.2.0 (10.8.2.0)
      Origin INCOMPLETE, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:1:10001 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:00:00:00:10:10:22
      VNI: 11001
BGP routing table entry for ip-prefix 0.0.0.0/0, Route Distinguisher: 10.9.0.5:2
 Paths: 2 available
  65200 65298 65301
    10.9.0.5 from 10.8.1.0 (10.8.1.0)
      Origin INCOMPLETE, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:00:00:00:10:10:22
      VNI: 11002
  65200 65298 65301
    10.9.0.5 from 10.8.2.0 (10.8.2.0)
      Origin INCOMPLETE, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:00:00:00:10:10:22
      VNI: 11002
BGP routing table entry for ip-prefix 0.0.0.0/0, Route Distinguisher: 10.9.0.5:3
 Paths: 2 available
  65200 65298 65301
    10.9.0.5 from 10.8.1.0 (10.8.1.0)
      Origin INCOMPLETE, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:3:10003 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:00:00:00:10:10:22
      VNI: 11003
  65200 65298 65301
    10.9.0.5 from 10.8.2.0 (10.8.2.0)
      Origin INCOMPLETE, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:3:10003 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:00:00:00:10:10:22
      VNI: 11003
BGP routing table entry for ip-prefix 0.0.0.0/0, Route Distinguisher: 10.9.0.6:1
 Paths: 2 available
  65200 65299 65301
    10.9.0.5 from 10.8.1.0 (10.8.1.0)
      Origin INCOMPLETE, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:1:10001 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:00:00:00:10:10:22
      VNI: 11001
  65200 65299 65301
    10.9.0.5 from 10.8.2.0 (10.8.2.0)
      Origin INCOMPLETE, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:1:10001 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:00:00:00:10:10:22
      VNI: 11001
BGP routing table entry for ip-prefix 0.0.0.0/0, Route Distinguisher: 10.9.0.6:2
 Paths: 2 available
  65200 65299 65301
    10.9.0.5 from 10.8.1.0 (10.8.1.0)
      Origin INCOMPLETE, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:00:00:00:10:10:22
      VNI: 11002
  65200 65299 65301
    10.9.0.5 from 10.8.2.0 (10.8.2.0)
      Origin INCOMPLETE, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:00:00:00:10:10:22
      VNI: 11002
BGP routing table entry for ip-prefix 0.0.0.0/0, Route Distinguisher: 10.9.0.6:3
 Paths: 2 available
  65200 65299 65301
    10.9.0.5 from 10.8.1.0 (10.8.1.0)
      Origin INCOMPLETE, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:3:10003 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:00:00:00:10:10:22
      VNI: 11003
  65200 65299 65301
    10.9.0.5 from 10.8.2.0 (10.8.2.0)
      Origin INCOMPLETE, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:3:10003 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:00:00:00:10:10:22
      VNI: 11003
DC2-Leaf-1-1#

```
</details>

<details>
<summary> 6.4 Проверка на DC1-Border-GW1 </summary>

```
DC1-Border-GW1#sh bgp summary
BGP summary information for VRF default
Router identifier 10.0.0.5, local AS number 65198
Neighbor            AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc
---------- ----------- ------------- ----------------------- -------------- ---------- ----------
10.0.1.0         65100 Established   L2VPN EVPN              Negotiated             23         23
10.0.2.0         65100 Active        L2VPN EVPN              Configured              0          0
10.2.1.8         65100 Established   IPv4 Unicast            Negotiated              7          7
10.2.2.8         65100 Active        IPv4 Unicast            Configured              0          0
10.2.91.0        65999 Established   IPv4 Unicast            Negotiated              5          5
10.2.92.0        65999 Established   IPv4 Unicast            Negotiated              5          5
10.91.91.1       65999 Established   L2VPN EVPN              Negotiated             35         35
10.91.91.2       65999 Established   L2VPN EVPN              Negotiated             35         35
DC1-Border-GW1#show ip bgp
BGP routing table information for VRF default
Router identifier 10.0.0.5, local AS number 65198
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.0.0.1/32            10.2.1.8              0       -          100     0       65100 65101 i
 * >      10.0.0.2/32            10.2.1.8              0       -          100     0       65100 65101 i
 * >      10.0.0.5/32            -                     -       -          -       0       i
 * >Ec    10.0.0.6/32            10.2.92.0             0       -          100     0       65999 65199 i
 *  ec    10.0.0.6/32            10.2.91.0             0       -          100     0       65999 65199 i
 *  ec    10.0.0.6/32            10.2.1.8              0       -          100     0       65100 65199 i
 * >      10.0.1.0/32            10.2.1.8              0       -          100     0       65100 i
 * >      10.1.0.1/32            10.2.1.8              0       -          100     0       65100 65101 i
 * >      10.1.0.2/32            10.2.1.8              0       -          100     0       65100 65101 i
 * >      10.1.0.5/32            -                     -       -          -       0       i
 * >      10.1.1.0/32            10.2.1.8              0       -          100     0       65100 i
 * >Ec    10.8.0.5/32            10.2.92.0             0       -          100     0       65999 65298 i
 *  ec    10.8.0.5/32            10.2.91.0             0       -          100     0       65999 65298 i
 * >Ec    10.8.0.6/32            10.2.91.0             0       -          100     0       65999 65299 i
 *  ec    10.8.0.6/32            10.2.92.0             0       -          100     0       65999 65299 i
 * >Ec    10.9.0.5/32            10.2.92.0             0       -          100     0       65999 65298 i
 *  ec    10.9.0.5/32            10.2.91.0             0       -          100     0       65999 65298 i
 * >      10.91.91.1/32          10.2.91.0             0       -          100     0       65999 i
 * >      10.91.91.2/32          10.2.92.0             0       -          100     0       65999 i
DC1-Border-GW1#
DC1-Border-GW1#show bgp evpn
BGP routing table information for VRF default
Router identifier 10.0.0.5, local AS number 65198
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 65101:10010 mac-ip aabb.cc80.6000
                                 10.1.0.1              -       100     0       65100 65101 i
 * >      RD: 65101:10010 mac-ip aabb.cc80.6000 10.4.10.11
                                 10.1.0.1              -       100     0       65100 65101 i
 * >      RD: 10.0.0.5:20010 mac-ip aabb.cc81.6000
                                 -                     -       100     0       65999 65298 65200 65201 i
          RD: 10.0.0.6:20010 mac-ip aabb.cc81.6000
                                 10.1.0.5              -       100     0       65100 65199 65999 65298 65200 65201 i
 * >      RD: 10.0.0.5:20010 mac-ip aabb.cc81.6000 10.4.10.21
                                 -                     -       100     0       65999 65298 65200 65201 i
          RD: 10.0.0.6:20010 mac-ip aabb.cc81.6000 10.4.10.21
                                 10.1.0.5              -       100     0       65100 65199 65999 65298 65200 65201 i
 * >      RD: 10.0.0.5:20010 mac-ip aabb.cc81.7000
                                 -                     -       100     0       65999 65298 65200 65202 i
          RD: 10.0.0.6:20010 mac-ip aabb.cc81.7000
                                 10.1.0.5              -       100     0       65100 65199 65999 65298 65200 65202 i
 * >      RD: 10.0.0.5:20010 mac-ip aabb.cc80.6000 remote
                                 -                     -       100     0       65100 65101 i
          RD: 10.0.0.6:20010 mac-ip aabb.cc80.6000 remote
                                 10.1.0.5              -       100     0       65999 65199 65100 65101 i
          RD: 10.0.0.6:20010 mac-ip aabb.cc80.6000 remote
                                 10.1.0.5              -       100     0       65999 65199 65100 65101 i
 * >      RD: 10.0.0.5:20010 mac-ip aabb.cc80.6000 10.4.10.11 remote
                                 -                     -       100     0       65100 65101 i
          RD: 10.0.0.6:20010 mac-ip aabb.cc80.6000 10.4.10.11 remote
                                 10.1.0.5              -       100     0       65999 65199 65100 65101 i
          RD: 10.0.0.6:20010 mac-ip aabb.cc80.6000 10.4.10.11 remote
                                 10.1.0.5              -       100     0       65999 65199 65100 65101 i
 * >Ec    RD: 10.8.0.5:20010 mac-ip aabb.cc81.6000 remote
                                 10.9.0.5              -       100     0       65999 65298 65200 65201 i
 *  ec    RD: 10.8.0.5:20010 mac-ip aabb.cc81.6000 remote
                                 10.9.0.5              -       100     0       65999 65298 65200 65201 i
 * >Ec    RD: 10.8.0.6:20010 mac-ip aabb.cc81.6000 remote
                                 10.9.0.5              -       100     0       65999 65299 65200 65201 i
 *  ec    RD: 10.8.0.6:20010 mac-ip aabb.cc81.6000 remote
                                 10.9.0.5              -       100     0       65999 65299 65200 65201 i
 * >Ec    RD: 10.8.0.5:20010 mac-ip aabb.cc81.6000 10.4.10.21 remote
                                 10.9.0.5              -       100     0       65999 65298 65200 65201 i
 *  ec    RD: 10.8.0.5:20010 mac-ip aabb.cc81.6000 10.4.10.21 remote
                                 10.9.0.5              -       100     0       65999 65298 65200 65201 i
 * >Ec    RD: 10.8.0.6:20010 mac-ip aabb.cc81.6000 10.4.10.21 remote
                                 10.9.0.5              -       100     0       65999 65299 65200 65201 i
 *  ec    RD: 10.8.0.6:20010 mac-ip aabb.cc81.6000 10.4.10.21 remote
                                 10.9.0.5              -       100     0       65999 65299 65200 65201 i
 * >Ec    RD: 10.8.0.5:20010 mac-ip aabb.cc81.7000 remote
                                 10.9.0.5              -       100     0       65999 65298 65200 65202 i
 *  ec    RD: 10.8.0.5:20010 mac-ip aabb.cc81.7000 remote
                                 10.9.0.5              -       100     0       65999 65298 65200 65202 i
 * >Ec    RD: 10.8.0.6:20010 mac-ip aabb.cc81.7000 remote
                                 10.9.0.5              -       100     0       65999 65299 65200 65202 i
 *  ec    RD: 10.8.0.6:20010 mac-ip aabb.cc81.7000 remote
                                 10.9.0.5              -       100     0       65999 65299 65200 65202 i
 * >      RD: 65101:10010 imet 10.1.0.1
                                 10.1.0.1              -       100     0       65100 65101 i
 * >      RD: 65101:10020 imet 10.1.0.1
                                 10.1.0.1              -       100     0       65100 65101 i
 * >      RD: 65101:10010 imet 10.1.0.2
                                 10.1.0.2              -       100     0       65100 65101 i
 * >      RD: 65101:10020 imet 10.1.0.2
                                 10.1.0.2              -       100     0       65100 65101 i
 * >      RD: 10.0.0.5:20010 imet 10.1.0.5
                                 -                     -       -       0       i
          RD: 10.0.0.6:20010 imet 10.1.0.5
                                 10.1.0.5              -       100     0       65100 65199 i
 * >      RD: 10.0.0.5:20010 imet 10.1.0.5 remote
                                 -                     -       -       0       i
          RD: 10.0.0.6:20010 imet 10.1.0.5 remote
                                 10.1.0.5              -       100     0       65999 65199 i
          RD: 10.0.0.6:20010 imet 10.1.0.5 remote
                                 10.1.0.5              -       100     0       65999 65199 i
 * >Ec    RD: 10.8.0.5:20010 imet 10.9.0.5 remote
                                 10.9.0.5              -       100     0       65999 65298 i
 *  ec    RD: 10.8.0.5:20010 imet 10.9.0.5 remote
                                 10.9.0.5              -       100     0       65999 65298 i
 * >Ec    RD: 10.8.0.6:20010 imet 10.9.0.5 remote
                                 10.9.0.5              -       100     0       65999 65299 i
 *  ec    RD: 10.8.0.6:20010 imet 10.9.0.5 remote
                                 10.9.0.5              -       100     0       65999 65299 i
 * >Ec    RD: 10.1.0.5:1 ip-prefix 0.0.0.0/0
                                 -                     -       100     0       65301 ?
 *  ec    RD: 10.1.0.5:1 ip-prefix 0.0.0.0/0
                                 -                     -       100     0       65301 ?
 * >Ec    RD: 10.1.0.5:2 ip-prefix 0.0.0.0/0
                                 -                     -       100     0       65301 ?
 *  ec    RD: 10.1.0.5:2 ip-prefix 0.0.0.0/0
                                 -                     -       100     0       65301 ?
          RD: 10.1.0.6:1 ip-prefix 0.0.0.0/0
                                 10.1.0.5              -       100     0       65100 65199 65301 ?
          RD: 10.1.0.6:2 ip-prefix 0.0.0.0/0
                                 10.1.0.5              -       100     0       65100 65199 65301 ?
 * >      RD: 10.1.0.5:1 ip-prefix 10.4.0.0/16
                                 -                     -       -       0       65100 65101 i
          RD: 10.1.0.6:1 ip-prefix 10.4.0.0/16
                                 10.1.0.5              -       100     0       65999 65199 65100 65101 i
          RD: 10.1.0.6:1 ip-prefix 10.4.0.0/16
                                 10.1.0.5              -       100     0       65999 65199 65100 65101 i
 * >Ec    RD: 10.9.0.5:1 ip-prefix 10.4.0.0/16
                                 10.9.0.5              -       100     0       65999 65298 65200 i
 *  ec    RD: 10.9.0.5:1 ip-prefix 10.4.0.0/16
                                 10.9.0.5              -       100     0       65999 65298 65200 i
 * >Ec    RD: 10.9.0.6:1 ip-prefix 10.4.0.0/16
                                 10.9.0.5              -       100     0       65999 65299 65200 65201 i
 *  ec    RD: 10.9.0.6:1 ip-prefix 10.4.0.0/16
                                 10.9.0.5              -       100     0       65999 65299 65200 65201 i
          RD: 10.9.0.6:1 ip-prefix 10.4.0.0/16
                                 10.1.0.5              -       100     0       65100 65199 65999 65299 65200 65201 i
 * >      RD: 10.1.0.1:1 ip-prefix 10.4.10.0/24
                                 10.1.0.1              -       100     0       65100 65101 i
 * >      RD: 10.1.0.2:1 ip-prefix 10.4.10.0/24
                                 10.1.0.2              -       100     0       65100 65101 i
 * >Ec    RD: 10.9.0.1:1 ip-prefix 10.4.10.0/24
                                 10.9.0.5              -       100     0       65999 65298 65200 65201 i
 *  ec    RD: 10.9.0.1:1 ip-prefix 10.4.10.0/24
                                 10.9.0.5              -       100     0       65999 65298 65200 65201 i
 * >Ec    RD: 10.9.0.2:1 ip-prefix 10.4.10.0/24
                                 10.9.0.5              -       100     0       65999 65298 65200 65201 i
 *  ec    RD: 10.9.0.2:1 ip-prefix 10.4.10.0/24
                                 10.9.0.5              -       100     0       65999 65298 65200 65201 i
 * >Ec    RD: 10.9.0.3:1 ip-prefix 10.4.10.0/24
                                 10.9.0.5              -       100     0       65999 65298 65200 65202 i
 *  ec    RD: 10.9.0.3:1 ip-prefix 10.4.10.0/24
                                 10.9.0.5              -       100     0       65999 65298 65200 65202 i
 * >Ec    RD: 10.9.0.4:1 ip-prefix 10.4.10.0/24
                                 10.9.0.5              -       100     0       65999 65298 65200 65202 i
 *  ec    RD: 10.9.0.4:1 ip-prefix 10.4.10.0/24
                                 10.9.0.5              -       100     0       65999 65298 65200 65202 i
 * >      RD: 10.1.0.5:2 ip-prefix 10.5.0.0/16
                                 -                     -       -       0       65100 65101 i
          RD: 10.1.0.6:2 ip-prefix 10.5.0.0/16
                                 10.1.0.5              -       100     0       65999 65199 65100 65101 i
          RD: 10.1.0.6:2 ip-prefix 10.5.0.0/16
                                 10.1.0.5              -       100     0       65999 65199 65100 65101 i
 * >      RD: 10.1.0.1:2 ip-prefix 10.5.20.0/24
                                 10.1.0.1              -       100     0       65100 65101 i
 * >      RD: 10.1.0.2:2 ip-prefix 10.5.20.0/24
                                 10.1.0.2              -       100     0       65100 65101 i
 * >      RD: 10.1.0.5:2 ip-prefix 10.12.0.0/16
                                 -                     -       -       0       65999 65298 65200 65201 i
          RD: 10.1.0.6:2 ip-prefix 10.12.0.0/16
                                 10.1.0.5              -       100     0       65100 65199 65999 65298 65200 65201 i
 * >Ec    RD: 10.9.0.5:2 ip-prefix 10.12.0.0/16
                                 10.9.0.5              -       100     0       65999 65298 65200 65201 i
 *  ec    RD: 10.9.0.5:2 ip-prefix 10.12.0.0/16
                                 10.9.0.5              -       100     0       65999 65298 65200 65201 i
          RD: 10.9.0.5:2 ip-prefix 10.12.0.0/16
                                 10.1.0.5              -       100     0       65100 65199 65999 65298 65200 65201 i
 * >Ec    RD: 10.9.0.6:2 ip-prefix 10.12.0.0/16
                                 10.9.0.5              -       100     0       65999 65299 65200 65201 i
 *  ec    RD: 10.9.0.6:2 ip-prefix 10.12.0.0/16
                                 10.9.0.5              -       100     0       65999 65299 65200 65201 i
          RD: 10.9.0.6:2 ip-prefix 10.12.0.0/16
                                 10.1.0.5              -       100     0       65100 65199 65999 65299 65200 65201 i
 * >Ec    RD: 10.9.0.1:2 ip-prefix 10.12.20.0/24
                                 10.9.0.5              -       100     0       65999 65298 65200 65201 i
 *  ec    RD: 10.9.0.1:2 ip-prefix 10.12.20.0/24
                                 10.9.0.5              -       100     0       65999 65298 65200 65201 i
 * >Ec    RD: 10.9.0.2:2 ip-prefix 10.12.20.0/24
                                 10.9.0.5              -       100     0       65999 65298 65200 65201 i
 *  ec    RD: 10.9.0.2:2 ip-prefix 10.12.20.0/24
                                 10.9.0.5              -       100     0       65999 65298 65200 65201 i
 * >Ec    RD: 10.9.0.3:2 ip-prefix 10.12.20.0/24
                                 10.9.0.5              -       100     0       65999 65298 65200 65202 i
 *  ec    RD: 10.9.0.3:2 ip-prefix 10.12.20.0/24
                                 10.9.0.5              -       100     0       65999 65298 65200 65202 i
 * >Ec    RD: 10.9.0.4:2 ip-prefix 10.12.20.0/24
                                 10.9.0.5              -       100     0       65999 65298 65200 65202 i
 *  ec    RD: 10.9.0.4:2 ip-prefix 10.12.20.0/24
                                 10.9.0.5              -       100     0       65999 65298 65200 65202 i
 * >      RD: 10.1.0.5:1 ip-prefix 10.14.0.0/16
                                 -                     -       -       0       65999 65298 65200 65201 i
          RD: 10.1.0.6:1 ip-prefix 10.14.0.0/16
                                 10.1.0.5              -       100     0       65100 65199 65999 65298 65200 65201 i
 * >Ec    RD: 10.9.0.5:1 ip-prefix 10.14.0.0/16
                                 10.9.0.5              -       100     0       65999 65298 65200 65201 i
 *  ec    RD: 10.9.0.5:1 ip-prefix 10.14.0.0/16
                                 10.9.0.5              -       100     0       65999 65298 65200 65201 i
          RD: 10.9.0.5:1 ip-prefix 10.14.0.0/16
                                 10.1.0.5              -       100     0       65100 65199 65999 65298 65200 65201 i
 * >Ec    RD: 10.9.0.6:1 ip-prefix 10.14.0.0/16
                                 10.9.0.5              -       100     0       65999 65299 65200 65201 i
 *  ec    RD: 10.9.0.6:1 ip-prefix 10.14.0.0/16
                                 10.9.0.5              -       100     0       65999 65299 65200 65201 i
          RD: 10.9.0.6:1 ip-prefix 10.14.0.0/16
                                 10.1.0.5              -       100     0       65100 65199 65999 65299 65200 65201 i
 * >Ec    RD: 10.9.0.1:1 ip-prefix 10.14.14.0/24
                                 10.9.0.5              -       100     0       65999 65298 65200 65201 i
 *  ec    RD: 10.9.0.1:1 ip-prefix 10.14.14.0/24
                                 10.9.0.5              -       100     0       65999 65298 65200 65201 i
 * >Ec    RD: 10.9.0.2:1 ip-prefix 10.14.14.0/24
                                 10.9.0.5              -       100     0       65999 65298 65200 65201 i
 *  ec    RD: 10.9.0.2:1 ip-prefix 10.14.14.0/24
                                 10.9.0.5              -       100     0       65999 65298 65200 65201 i
          RD: 10.1.0.6:1 ip-prefix 0.0.0.0/0 remote
                                 10.1.0.5              -       100     0       65100 65199 65301 ?
          RD: 10.1.0.6:2 ip-prefix 0.0.0.0/0 remote
                                 10.1.0.5              -       100     0       65100 65199 65301 ?
 * >      RD: 10.1.0.5:1 ip-prefix 10.4.0.0/16 remote
                                 -                     -       -       0       65100 65101 i
          RD: 10.1.0.6:1 ip-prefix 10.4.0.0/16 remote
                                 10.1.0.5              -       100     0       65999 65199 65100 65101 i
          RD: 10.1.0.6:1 ip-prefix 10.4.0.0/16 remote
                                 10.1.0.5              -       100     0       65999 65199 65100 65101 i
 * >Ec    RD: 10.9.0.5:1 ip-prefix 10.4.0.0/16 remote
                                 10.9.0.5              -       100     0       65999 65298 65200 i
 *  ec    RD: 10.9.0.5:1 ip-prefix 10.4.0.0/16 remote
                                 10.9.0.5              -       100     0       65999 65298 65200 i
 * >Ec    RD: 10.9.0.6:1 ip-prefix 10.4.0.0/16 remote
                                 10.9.0.5              -       100     0       65999 65299 65200 65201 i
 *  ec    RD: 10.9.0.6:1 ip-prefix 10.4.0.0/16 remote
                                 10.9.0.5              -       100     0       65999 65299 65200 65201 i
          RD: 10.9.0.6:1 ip-prefix 10.4.0.0/16 remote
                                 10.1.0.5              -       100     0       65100 65199 65999 65299 65200 65201 i
 * >      RD: 10.1.0.1:1 ip-prefix 10.4.10.0/24 remote
                                 10.1.0.1              -       100     0       65100 65101 i
 * >      RD: 10.1.0.2:1 ip-prefix 10.4.10.0/24 remote
                                 10.1.0.2              -       100     0       65100 65101 i
 * >Ec    RD: 10.9.0.1:1 ip-prefix 10.4.10.0/24 remote
                                 10.9.0.5              -       100     0       65999 65298 65200 65201 i
 *  ec    RD: 10.9.0.1:1 ip-prefix 10.4.10.0/24 remote
                                 10.9.0.5              -       100     0       65999 65298 65200 65201 i
 * >Ec    RD: 10.9.0.2:1 ip-prefix 10.4.10.0/24 remote
                                 10.9.0.5              -       100     0       65999 65298 65200 65201 i
 *  ec    RD: 10.9.0.2:1 ip-prefix 10.4.10.0/24 remote
                                 10.9.0.5              -       100     0       65999 65298 65200 65201 i
 * >Ec    RD: 10.9.0.3:1 ip-prefix 10.4.10.0/24 remote
                                 10.9.0.5              -       100     0       65999 65298 65200 65202 i
 *  ec    RD: 10.9.0.3:1 ip-prefix 10.4.10.0/24 remote
                                 10.9.0.5              -       100     0       65999 65298 65200 65202 i
 * >Ec    RD: 10.9.0.4:1 ip-prefix 10.4.10.0/24 remote
                                 10.9.0.5              -       100     0       65999 65298 65200 65202 i
 *  ec    RD: 10.9.0.4:1 ip-prefix 10.4.10.0/24 remote
                                 10.9.0.5              -       100     0       65999 65298 65200 65202 i
 * >      RD: 10.1.0.5:2 ip-prefix 10.5.0.0/16 remote
                                 -                     -       -       0       65100 65101 i
          RD: 10.1.0.6:2 ip-prefix 10.5.0.0/16 remote
                                 10.1.0.5              -       100     0       65999 65199 65100 65101 i
          RD: 10.1.0.6:2 ip-prefix 10.5.0.0/16 remote
                                 10.1.0.5              -       100     0       65999 65199 65100 65101 i
 * >      RD: 10.1.0.1:2 ip-prefix 10.5.20.0/24 remote
                                 10.1.0.1              -       100     0       65100 65101 i
 * >      RD: 10.1.0.2:2 ip-prefix 10.5.20.0/24 remote
                                 10.1.0.2              -       100     0       65100 65101 i
 * >      RD: 10.1.0.5:2 ip-prefix 10.12.0.0/16 remote
                                 -                     -       -       0       65999 65298 65200 65201 i
          RD: 10.1.0.6:2 ip-prefix 10.12.0.0/16 remote
                                 10.1.0.5              -       100     0       65100 65199 65999 65298 65200 65201 i
 * >Ec    RD: 10.9.0.5:2 ip-prefix 10.12.0.0/16 remote
                                 10.9.0.5              -       100     0       65999 65298 65200 65201 i
 *  ec    RD: 10.9.0.5:2 ip-prefix 10.12.0.0/16 remote
                                 10.9.0.5              -       100     0       65999 65298 65200 65201 i
          RD: 10.9.0.5:2 ip-prefix 10.12.0.0/16 remote
                                 10.1.0.5              -       100     0       65100 65199 65999 65298 65200 65201 i
 * >Ec    RD: 10.9.0.6:2 ip-prefix 10.12.0.0/16 remote
                                 10.9.0.5              -       100     0       65999 65299 65200 65201 i
 *  ec    RD: 10.9.0.6:2 ip-prefix 10.12.0.0/16 remote
                                 10.9.0.5              -       100     0       65999 65299 65200 65201 i
          RD: 10.9.0.6:2 ip-prefix 10.12.0.0/16 remote
                                 10.1.0.5              -       100     0       65100 65199 65999 65299 65200 65201 i
 * >Ec    RD: 10.9.0.1:2 ip-prefix 10.12.20.0/24 remote
                                 10.9.0.5              -       100     0       65999 65298 65200 65201 i
 *  ec    RD: 10.9.0.1:2 ip-prefix 10.12.20.0/24 remote
                                 10.9.0.5              -       100     0       65999 65298 65200 65201 i
 * >Ec    RD: 10.9.0.2:2 ip-prefix 10.12.20.0/24 remote
                                 10.9.0.5              -       100     0       65999 65298 65200 65201 i
 *  ec    RD: 10.9.0.2:2 ip-prefix 10.12.20.0/24 remote
                                 10.9.0.5              -       100     0       65999 65298 65200 65201 i
 * >Ec    RD: 10.9.0.3:2 ip-prefix 10.12.20.0/24 remote
                                 10.9.0.5              -       100     0       65999 65298 65200 65202 i
 *  ec    RD: 10.9.0.3:2 ip-prefix 10.12.20.0/24 remote
                                 10.9.0.5              -       100     0       65999 65298 65200 65202 i
 * >Ec    RD: 10.9.0.4:2 ip-prefix 10.12.20.0/24 remote
                                 10.9.0.5              -       100     0       65999 65298 65200 65202 i
 *  ec    RD: 10.9.0.4:2 ip-prefix 10.12.20.0/24 remote
                                 10.9.0.5              -       100     0       65999 65298 65200 65202 i
 * >Ec    RD: 10.9.0.5:3 ip-prefix 10.13.0.0/16 remote
                                 10.9.0.5              -       100     0       65999 65298 65200 i
 *  ec    RD: 10.9.0.5:3 ip-prefix 10.13.0.0/16 remote
                                 10.9.0.5              -       100     0       65999 65298 65200 i
 * >Ec    RD: 10.9.0.6:3 ip-prefix 10.13.0.0/16 remote
                                 10.9.0.5              -       100     0       65999 65299 65200 i
 *  ec    RD: 10.9.0.6:3 ip-prefix 10.13.0.0/16 remote
                                 10.9.0.5              -       100     0       65999 65299 65200 i
 * >Ec    RD: 10.9.0.1:3 ip-prefix 10.13.30.0/24 remote
                                 10.9.0.5              -       100     0       65999 65298 65200 65201 i
 *  ec    RD: 10.9.0.1:3 ip-prefix 10.13.30.0/24 remote
                                 10.9.0.5              -       100     0       65999 65298 65200 65201 i
 * >Ec    RD: 10.9.0.2:3 ip-prefix 10.13.30.0/24 remote
                                 10.9.0.5              -       100     0       65999 65298 65200 65201 i
 *  ec    RD: 10.9.0.2:3 ip-prefix 10.13.30.0/24 remote
                                 10.9.0.5              -       100     0       65999 65298 65200 65201 i
 * >Ec    RD: 10.9.0.3:3 ip-prefix 10.13.30.0/24 remote
                                 10.9.0.5              -       100     0       65999 65298 65200 65202 i
 *  ec    RD: 10.9.0.3:3 ip-prefix 10.13.30.0/24 remote
                                 10.9.0.5              -       100     0       65999 65298 65200 65202 i
 * >Ec    RD: 10.9.0.4:3 ip-prefix 10.13.30.0/24 remote
                                 10.9.0.5              -       100     0       65999 65298 65200 65202 i
 *  ec    RD: 10.9.0.4:3 ip-prefix 10.13.30.0/24 remote
                                 10.9.0.5              -       100     0       65999 65298 65200 65202 i
 * >      RD: 10.1.0.5:1 ip-prefix 10.14.0.0/16 remote
                                 -                     -       -       0       65999 65298 65200 65201 i
          RD: 10.1.0.6:1 ip-prefix 10.14.0.0/16 remote
                                 10.1.0.5              -       100     0       65100 65199 65999 65298 65200 65201 i
 * >Ec    RD: 10.9.0.5:1 ip-prefix 10.14.0.0/16 remote
                                 10.9.0.5              -       100     0       65999 65298 65200 65201 i
 *  ec    RD: 10.9.0.5:1 ip-prefix 10.14.0.0/16 remote
                                 10.9.0.5              -       100     0       65999 65298 65200 65201 i
          RD: 10.9.0.5:1 ip-prefix 10.14.0.0/16 remote
                                 10.1.0.5              -       100     0       65100 65199 65999 65298 65200 65201 i
 * >Ec    RD: 10.9.0.6:1 ip-prefix 10.14.0.0/16 remote
                                 10.9.0.5              -       100     0       65999 65299 65200 65201 i
 *  ec    RD: 10.9.0.6:1 ip-prefix 10.14.0.0/16 remote
                                 10.9.0.5              -       100     0       65999 65299 65200 65201 i
          RD: 10.9.0.6:1 ip-prefix 10.14.0.0/16 remote
                                 10.1.0.5              -       100     0       65100 65199 65999 65299 65200 65201 i
 * >Ec    RD: 10.9.0.1:1 ip-prefix 10.14.14.0/24 remote
                                 10.9.0.5              -       100     0       65999 65298 65200 65201 i
 *  ec    RD: 10.9.0.1:1 ip-prefix 10.14.14.0/24 remote
                                 10.9.0.5              -       100     0       65999 65298 65200 65201 i
 * >Ec    RD: 10.9.0.2:1 ip-prefix 10.14.14.0/24 remote
                                 10.9.0.5              -       100     0       65999 65298 65200 65201 i
 *  ec    RD: 10.9.0.2:1 ip-prefix 10.14.14.0/24 remote
                                 10.9.0.5              -       100     0       65999 65298 65200 65201 i
DC1-Border-GW1#
DC1-Border-GW1#
DC1-Border-GW1#show ip route vrf BLUE

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
 B E      0.0.0.0/0 [200/0] via 10.198.104.2, Ethernet3.104
                            via 10.198.104.10, Ethernet6.114

 B E      10.4.10.11/32 [200/0] via VTEP 10.1.0.1 VNI 11001 router-mac 50:00:00:d5:5d:c0 local-interface Vxlan1
 B E      10.4.10.0/24 [200/0] via VTEP 10.1.0.2 VNI 11001 router-mac 50:00:00:88:fe:27 local-interface Vxlan1
                               via VTEP 10.1.0.1 VNI 11001 router-mac 50:00:00:d5:5d:c0 local-interface Vxlan1
 B E      10.14.14.0/24 [200/0] via VTEP 10.9.0.5 VNI 11001 router-mac 00:00:00:10:10:22 local-interface Vxlan1
 C        10.198.104.0/30 is directly connected, Ethernet3.104
 C        10.198.104.8/30 is directly connected, Ethernet6.114

DC1-Border-GW1#show ip route vrf RED

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
 B E      0.0.0.0/0 [200/0] via 10.198.105.2, Ethernet3.105
                            via 10.198.105.10, Ethernet6.115

 B E      10.5.20.0/24 [200/0] via VTEP 10.1.0.1 VNI 11002 router-mac 50:00:00:d5:5d:c0 local-interface Vxlan1
                               via VTEP 10.1.0.2 VNI 11002 router-mac 50:00:00:88:fe:27 local-interface Vxlan1
 B E      10.12.20.0/24 [200/0] via VTEP 10.9.0.5 VNI 11002 router-mac 00:00:00:10:10:22 local-interface Vxlan1
 C        10.198.105.0/30 is directly connected, Ethernet3.105
 C        10.198.105.8/30 is directly connected, Ethernet6.115

DC1-Border-GW1#show ip route vrf GREEN

VRF: GREEN
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
 B E      0.0.0.0/0 [200/0] via 10.198.106.2, Ethernet3.106
                            via 10.198.106.10, Ethernet6.116

 B E      10.13.30.0/24 [200/0] via VTEP 10.9.0.5 VNI 11003 router-mac 00:00:00:10:10:22 local-interface Vxlan1
 C        10.198.106.0/30 is directly connected, Ethernet3.106
 C        10.198.106.8/30 is directly connected, Ethernet6.116

DC1-Border-GW1#show mac address-table
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
  10    0000.0000.0001    STATIC      Cpu
  10    aabb.cc80.6000    DYNAMIC     Vx1        1       0:06:46 ago
  10    aabb.cc81.6000    DYNAMIC     Vx1        1       0:05:59 ago
  10    aabb.cc81.7000    DYNAMIC     Vx1        1       22:46:44 ago
4092    0000.0000.0001    STATIC      Cpu
4092    0000.0010.1022    DYNAMIC     Vx1        1       22:46:50 ago
4093    0000.0000.0001    STATIC      Cpu
4093    0000.0010.1022    DYNAMIC     Vx1        1       22:46:45 ago
4093    5000.0088.fe27    DYNAMIC     Vx1        1       22:51:15 ago
4093    5000.00d5.5dc0    DYNAMIC     Vx1        1       22:51:15 ago
4094    0000.0000.0001    STATIC      Cpu
4094    0000.0010.1022    DYNAMIC     Vx1        1       22:46:45 ago
4094    5000.0088.fe27    DYNAMIC     Vx1        1       22:51:15 ago
4094    5000.00d5.5dc0    DYNAMIC     Vx1        1       22:51:14 ago
Total Mac Addresses for this criterion: 14

          Multicast Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       ----        -----
Total Mac Addresses for this criterion: 0

DC1-Border-GW1#show vxlan address-table
          Vxlan Mac Address Table
----------------------------------------------------------------------

VLAN  Mac Address     Type      Prt  VTEP             Moves   Last Move
----  -----------     ----      ---  ----             -----   ---------
  10  aabb.cc80.6000  EVPN      Vx1  10.1.0.1         1       0:07:37 ago
  10  aabb.cc81.6000  EVPN      Vx1  10.9.0.5         1       0:06:51 ago
  10  aabb.cc81.7000  EVPN      Vx1  10.9.0.5         1       22:47:36 ago
4092  0000.0010.1022  EVPN      Vx1  10.9.0.5         1       22:47:41 ago
4093  0000.0010.1022  EVPN      Vx1  10.9.0.5         1       22:47:37 ago
4093  5000.0088.fe27  EVPN      Vx1  10.1.0.2         1       22:52:07 ago
4093  5000.00d5.5dc0  EVPN      Vx1  10.1.0.1         1       22:52:07 ago
4094  0000.0010.1022  EVPN      Vx1  10.9.0.5         1       22:47:37 ago
4094  5000.0088.fe27  EVPN      Vx1  10.1.0.2         1       22:52:07 ago
4094  5000.00d5.5dc0  EVPN      Vx1  10.1.0.1         1       22:52:06 ago
Total Remote Mac Addresses for this criterion: 10
DC1-Border-GW1#
DC1-Border-GW1#
DC1-Border-GW1#show vxlan flood vtep
          VXLAN Flood VTEP Table
--------------------------------------------------------------------------------

VLANS                            Ip Address
-----------------------------   ------------------------------------------------
10                              10.1.0.1        10.1.0.2        10.9.0.5
DC1-Border-GW1#
DC1-Border-GW1#show vxlan vtep
Remote VTEPS for Vxlan1:

VTEP           Tunnel Type(s)
-------------- --------------
10.1.0.1       unicast, flood
10.1.0.2       unicast, flood
10.9.0.5       unicast, flood

Total number of remote VTEPS:  3
DC1-Border-GW1#
DC1-Border-GW1#show vxlan vni
VNI to VLAN Mapping for Vxlan1
VNI         VLAN       Source       Interface       802.1Q Tag
----------- ---------- ------------ --------------- ----------
10010       10         static       Vxlan1          10

VNI to dynamic VLAN Mapping for Vxlan1
VNI         VLAN       VRF         Source
----------- ---------- ----------- ------------
11001       4094       BLUE        evpn
11002       4093       RED         evpn
11003       4092       GREEN       evpn

DC1-Border-GW1#

```
</details>

<details>
<summary> 6.5 Проверка на DC2-Border-GW1 </summary>

```

DC2-Border-GW1#sh bgp summary
BGP summary information for VRF default
Router identifier 10.8.0.5, local AS number 65298
Neighbor            AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc
---------- ----------- ------------- ----------------------- -------------- ---------- ----------
10.2.91.4        65999 Established   IPv4 Unicast            Negotiated              5          5
10.2.92.4        65999 Established   IPv4 Unicast            Negotiated              5          5
10.8.1.0         65200 Established   L2VPN EVPN              Negotiated             56         56
10.8.2.0         65200 Established   L2VPN EVPN              Negotiated             56         56
10.10.1.8        65200 Established   IPv4 Unicast            Negotiated             11         11
10.10.2.8        65200 Established   IPv4 Unicast            Negotiated             11         11
10.91.91.1       65999 Established   L2VPN EVPN              Negotiated             22         22
10.91.91.2       65999 Established   L2VPN EVPN              Negotiated             22         22
DC2-Border-GW1#show ip bgp
BGP routing table information for VRF default
Router identifier 10.8.0.5, local AS number 65298
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >Ec    10.0.0.5/32            10.2.92.4             0       -          100     0       65999 65198 i
 *  ec    10.0.0.5/32            10.2.91.4             0       -          100     0       65999 65198 i
 * >Ec    10.0.0.6/32            10.2.92.4             0       -          100     0       65999 65199 i
 *  ec    10.0.0.6/32            10.2.91.4             0       -          100     0       65999 65199 i
 * >Ec    10.1.0.5/32            10.2.92.4             0       -          100     0       65999 65198 i
 *  ec    10.1.0.5/32            10.2.91.4             0       -          100     0       65999 65198 i
 * >Ec    10.8.0.1/32            10.10.1.8             0       -          100     0       65200 65201 i
 *  ec    10.8.0.1/32            10.10.2.8             0       -          100     0       65200 65201 i
 * >Ec    10.8.0.2/32            10.10.1.8             0       -          100     0       65200 65201 i
 *  ec    10.8.0.2/32            10.10.2.8             0       -          100     0       65200 65201 i
 * >Ec    10.8.0.3/32            10.10.1.8             0       -          100     0       65200 65202 i
 *  ec    10.8.0.3/32            10.10.2.8             0       -          100     0       65200 65202 i
 * >Ec    10.8.0.4/32            10.10.1.8             0       -          100     0       65200 65202 i
 *  ec    10.8.0.4/32            10.10.2.8             0       -          100     0       65200 65202 i
 * >      10.8.0.5/32            -                     -       -          -       0       i
 * >Ec    10.8.0.6/32            10.2.91.4             0       -          100     0       65999 65299 i
 *  ec    10.8.0.6/32            10.10.1.8             0       -          100     0       65200 65299 i
 *  ec    10.8.0.6/32            10.10.2.8             0       -          100     0       65200 65299 i
 *  ec    10.8.0.6/32            10.2.92.4             0       -          100     0       65999 65299 i
 * >      10.8.1.0/32            10.10.1.8             0       -          100     0       65200 i
 * >      10.8.2.0/32            10.10.2.8             0       -          100     0       65200 i
 * >Ec    10.9.0.1/32            10.10.1.8             0       -          100     0       65200 65201 i
 *  ec    10.9.0.1/32            10.10.2.8             0       -          100     0       65200 65201 i
 * >Ec    10.9.0.2/32            10.10.1.8             0       -          100     0       65200 65201 i
 *  ec    10.9.0.2/32            10.10.2.8             0       -          100     0       65200 65201 i
 * >Ec    10.9.0.3/32            10.10.1.8             0       -          100     0       65200 65202 i
 *  ec    10.9.0.3/32            10.10.2.8             0       -          100     0       65200 65202 i
 * >Ec    10.9.0.4/32            10.10.1.8             0       -          100     0       65200 65202 i
 *  ec    10.9.0.4/32            10.10.2.8             0       -          100     0       65200 65202 i
 * >      10.9.0.5/32            -                     -       -          -       0       i
 * >      10.9.1.0/32            10.10.1.8             0       -          100     0       65200 i
 * >      10.9.2.0/32            10.10.2.8             0       -          100     0       65200 i
 * >      10.91.91.1/32          10.2.91.4             0       -          100     0       65999 i
 * >      10.91.91.2/32          10.2.92.4             0       -          100     0       65999 i
DC2-Border-GW1#show bgp evpn
BGP routing table information for VRF default
Router identifier 10.8.0.5, local AS number 65298
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.8.0.5:20010 mac-ip aabb.cc80.6000
                                 -                     -       100     0       65999 65199 65100 65101 i
          RD: 10.8.0.6:20010 mac-ip aabb.cc80.6000
                                 10.9.0.5              -       100     0       65200 65299 65999 65199 65100 65101 i
          RD: 10.8.0.6:20010 mac-ip aabb.cc80.6000
                                 10.9.0.5              -       100     0       65200 65299 65999 65199 65100 65101 i
 * >      RD: 10.8.0.5:20010 mac-ip aabb.cc80.6000 10.4.10.11
                                 -                     -       100     0       65999 65198 65100 65101 i
          RD: 10.8.0.6:20010 mac-ip aabb.cc80.6000 10.4.10.11
                                 10.9.0.5              -       100     0       65200 65299 65999 65198 65100 65101 i
          RD: 10.8.0.6:20010 mac-ip aabb.cc80.6000 10.4.10.11
                                 10.9.0.5              -       100     0       65200 65299 65999 65198 65100 65101 i
 * >Ec    RD: 10.9.0.1:10010 mac-ip aabb.cc81.6000
                                 10.9.0.1              -       100     0       65200 65201 i
 *  ec    RD: 10.9.0.1:10010 mac-ip aabb.cc81.6000
                                 10.9.0.1              -       100     0       65200 65201 i
 * >Ec    RD: 10.9.0.2:10010 mac-ip aabb.cc81.6000
                                 10.9.0.2              -       100     0       65200 65201 i
 *  ec    RD: 10.9.0.2:10010 mac-ip aabb.cc81.6000
                                 10.9.0.2              -       100     0       65200 65201 i
 * >Ec    RD: 10.9.0.1:10010 mac-ip aabb.cc81.6000 10.4.10.21
                                 10.9.0.1              -       100     0       65200 65201 i
 *  ec    RD: 10.9.0.1:10010 mac-ip aabb.cc81.6000 10.4.10.21
                                 10.9.0.1              -       100     0       65200 65201 i
 * >Ec    RD: 10.9.0.2:10010 mac-ip aabb.cc81.6000 10.4.10.21
                                 10.9.0.2              -       100     0       65200 65201 i
 *  ec    RD: 10.9.0.2:10010 mac-ip aabb.cc81.6000 10.4.10.21
                                 10.9.0.2              -       100     0       65200 65201 i
 * >Ec    RD: 10.9.0.3:10030 mac-ip aabb.cc81.7000
                                 10.9.0.3              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.3:10030 mac-ip aabb.cc81.7000
                                 10.9.0.3              -       100     0       65200 65202 i
 * >Ec    RD: 10.9.0.4:10010 mac-ip aabb.cc81.7000
                                 10.9.0.4              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.4:10010 mac-ip aabb.cc81.7000
                                 10.9.0.4              -       100     0       65200 65202 i
 * >Ec    RD: 10.9.0.4:10030 mac-ip aabb.cc81.7000
                                 10.9.0.4              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.4:10030 mac-ip aabb.cc81.7000
                                 10.9.0.4              -       100     0       65200 65202 i
 * >Ec    RD: 10.9.0.3:10030 mac-ip aabb.cc81.7000 10.13.30.23
                                 10.9.0.3              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.3:10030 mac-ip aabb.cc81.7000 10.13.30.23
                                 10.9.0.3              -       100     0       65200 65202 i
 * >Ec    RD: 10.9.0.4:10030 mac-ip aabb.cc81.7000 10.13.30.23
                                 10.9.0.4              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.4:10030 mac-ip aabb.cc81.7000 10.13.30.23
                                 10.9.0.4              -       100     0       65200 65202 i
 * >Ec    RD: 10.9.0.3:10020 mac-ip aabb.cc81.b000
                                 10.9.0.3              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.3:10020 mac-ip aabb.cc81.b000
                                 10.9.0.3              -       100     0       65200 65202 i
 * >Ec    RD: 10.9.0.4:10020 mac-ip aabb.cc81.b000
                                 10.9.0.4              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.4:10020 mac-ip aabb.cc81.b000
                                 10.9.0.4              -       100     0       65200 65202 i
 * >Ec    RD: 10.9.0.3:10020 mac-ip aabb.cc81.b000 10.12.20.24
                                 10.9.0.3              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.3:10020 mac-ip aabb.cc81.b000 10.12.20.24
                                 10.9.0.3              -       100     0       65200 65202 i
 * >Ec    RD: 10.9.0.4:10020 mac-ip aabb.cc81.b000 10.12.20.24
                                 10.9.0.4              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.4:10020 mac-ip aabb.cc81.b000 10.12.20.24
                                 10.9.0.4              -       100     0       65200 65202 i
 * >Ec    RD: 10.9.0.1:10030 mac-ip aabb.cc81.c000
                                 10.9.0.1              -       100     0       65200 65201 i
 *  ec    RD: 10.9.0.1:10030 mac-ip aabb.cc81.c000
                                 10.9.0.1              -       100     0       65200 65201 i
 * >Ec    RD: 10.9.0.2:10030 mac-ip aabb.cc81.c000
                                 10.9.0.2              -       100     0       65200 65201 i
 *  ec    RD: 10.9.0.2:10030 mac-ip aabb.cc81.c000
                                 10.9.0.2              -       100     0       65200 65201 i
 * >Ec    RD: 10.9.0.1:10030 mac-ip aabb.cc81.c000 10.13.30.22
                                 10.9.0.1              -       100     0       65200 65201 i
 *  ec    RD: 10.9.0.1:10030 mac-ip aabb.cc81.c000 10.13.30.22
                                 10.9.0.1              -       100     0       65200 65201 i
 * >Ec    RD: 10.9.0.2:10030 mac-ip aabb.cc81.c000 10.13.30.22
                                 10.9.0.2              -       100     0       65200 65201 i
 *  ec    RD: 10.9.0.2:10030 mac-ip aabb.cc81.c000 10.13.30.22
                                 10.9.0.2              -       100     0       65200 65201 i
 * >Ec    RD: 10.0.0.5:20010 mac-ip aabb.cc80.6000 remote
                                 10.1.0.5              -       100     0       65999 65198 65100 65101 i
 *  ec    RD: 10.0.0.5:20010 mac-ip aabb.cc80.6000 remote
                                 10.1.0.5              -       100     0       65999 65198 65100 65101 i
 * >Ec    RD: 10.0.0.6:20010 mac-ip aabb.cc80.6000 remote
                                 10.1.0.5              -       100     0       65999 65199 65100 65101 i
 *  ec    RD: 10.0.0.6:20010 mac-ip aabb.cc80.6000 remote
                                 10.1.0.5              -       100     0       65999 65199 65100 65101 i
 * >Ec    RD: 10.0.0.5:20010 mac-ip aabb.cc80.6000 10.4.10.11 remote
                                 10.1.0.5              -       100     0       65999 65198 65100 65101 i
 *  ec    RD: 10.0.0.5:20010 mac-ip aabb.cc80.6000 10.4.10.11 remote
                                 10.1.0.5              -       100     0       65999 65198 65100 65101 i
 * >Ec    RD: 10.0.0.6:20010 mac-ip aabb.cc80.6000 10.4.10.11 remote
                                 10.1.0.5              -       100     0       65999 65199 65100 65101 i
 *  ec    RD: 10.0.0.6:20010 mac-ip aabb.cc80.6000 10.4.10.11 remote
                                 10.1.0.5              -       100     0       65999 65199 65100 65101 i
 * >      RD: 10.8.0.5:20010 mac-ip aabb.cc81.6000 remote
                                 -                     -       100     0       65200 65201 i
          RD: 10.8.0.6:20010 mac-ip aabb.cc81.6000 remote
                                 10.9.0.5              -       100     0       65999 65299 65200 65201 i
          RD: 10.8.0.6:20010 mac-ip aabb.cc81.6000 remote
                                 10.9.0.5              -       100     0       65999 65299 65200 65201 i
 * >      RD: 10.8.0.5:20010 mac-ip aabb.cc81.6000 10.4.10.21 remote
                                 -                     -       100     0       65200 65201 i
          RD: 10.8.0.6:20010 mac-ip aabb.cc81.6000 10.4.10.21 remote
                                 10.9.0.5              -       100     0       65999 65299 65200 65201 i
          RD: 10.8.0.6:20010 mac-ip aabb.cc81.6000 10.4.10.21 remote
                                 10.9.0.5              -       100     0       65999 65299 65200 65201 i
 * >      RD: 10.8.0.5:20010 mac-ip aabb.cc81.7000 remote
                                 -                     -       100     0       65200 65202 i
          RD: 10.8.0.6:20010 mac-ip aabb.cc81.7000 remote
                                 10.9.0.5              -       100     0       65999 65299 65200 65202 i
          RD: 10.8.0.6:20010 mac-ip aabb.cc81.7000 remote
                                 10.9.0.5              -       100     0       65999 65299 65200 65202 i
 * >Ec    RD: 10.9.0.1:10010 imet 10.9.0.1
                                 10.9.0.1              -       100     0       65200 65201 i
 *  ec    RD: 10.9.0.1:10010 imet 10.9.0.1
                                 10.9.0.1              -       100     0       65200 65201 i
 * >Ec    RD: 10.9.0.1:10020 imet 10.9.0.1
                                 10.9.0.1              -       100     0       65200 65201 i
 *  ec    RD: 10.9.0.1:10020 imet 10.9.0.1
                                 10.9.0.1              -       100     0       65200 65201 i
 * >Ec    RD: 10.9.0.1:10030 imet 10.9.0.1
                                 10.9.0.1              -       100     0       65200 65201 i
 *  ec    RD: 10.9.0.1:10030 imet 10.9.0.1
                                 10.9.0.1              -       100     0       65200 65201 i
 * >Ec    RD: 10.9.0.2:10010 imet 10.9.0.2
                                 10.9.0.2              -       100     0       65200 65201 i
 *  ec    RD: 10.9.0.2:10010 imet 10.9.0.2
                                 10.9.0.2              -       100     0       65200 65201 i
 * >Ec    RD: 10.9.0.2:10020 imet 10.9.0.2
                                 10.9.0.2              -       100     0       65200 65201 i
 *  ec    RD: 10.9.0.2:10020 imet 10.9.0.2
                                 10.9.0.2              -       100     0       65200 65201 i
 * >Ec    RD: 10.9.0.2:10030 imet 10.9.0.2
                                 10.9.0.2              -       100     0       65200 65201 i
 *  ec    RD: 10.9.0.2:10030 imet 10.9.0.2
                                 10.9.0.2              -       100     0       65200 65201 i
 * >Ec    RD: 10.9.0.3:10010 imet 10.9.0.3
                                 10.9.0.3              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.3:10010 imet 10.9.0.3
                                 10.9.0.3              -       100     0       65200 65202 i
 * >Ec    RD: 10.9.0.3:10020 imet 10.9.0.3
                                 10.9.0.3              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.3:10020 imet 10.9.0.3
                                 10.9.0.3              -       100     0       65200 65202 i
 * >Ec    RD: 10.9.0.3:10030 imet 10.9.0.3
                                 10.9.0.3              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.3:10030 imet 10.9.0.3
                                 10.9.0.3              -       100     0       65200 65202 i
 * >Ec    RD: 10.9.0.4:10010 imet 10.9.0.4
                                 10.9.0.4              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.4:10010 imet 10.9.0.4
                                 10.9.0.4              -       100     0       65200 65202 i
 * >Ec    RD: 10.9.0.4:10020 imet 10.9.0.4
                                 10.9.0.4              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.4:10020 imet 10.9.0.4
                                 10.9.0.4              -       100     0       65200 65202 i
 * >Ec    RD: 10.9.0.4:10030 imet 10.9.0.4
                                 10.9.0.4              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.4:10030 imet 10.9.0.4
                                 10.9.0.4              -       100     0       65200 65202 i
 * >      RD: 10.8.0.5:20010 imet 10.9.0.5
                                 -                     -       -       0       i
          RD: 10.8.0.6:20010 imet 10.9.0.5
                                 10.9.0.5              -       100     0       65200 65299 i
          RD: 10.8.0.6:20010 imet 10.9.0.5
                                 10.9.0.5              -       100     0       65200 65299 i
 * >Ec    RD: 10.0.0.5:20010 imet 10.1.0.5 remote
                                 10.1.0.5              -       100     0       65999 65198 i
 *  ec    RD: 10.0.0.5:20010 imet 10.1.0.5 remote
                                 10.1.0.5              -       100     0       65999 65198 i
 * >Ec    RD: 10.0.0.6:20010 imet 10.1.0.5 remote
                                 10.1.0.5              -       100     0       65999 65199 i
 *  ec    RD: 10.0.0.6:20010 imet 10.1.0.5 remote
                                 10.1.0.5              -       100     0       65999 65199 i
 * >      RD: 10.8.0.5:20010 imet 10.9.0.5 remote
                                 -                     -       -       0       i
          RD: 10.8.0.6:20010 imet 10.9.0.5 remote
                                 10.9.0.5              -       100     0       65999 65299 i
          RD: 10.8.0.6:20010 imet 10.9.0.5 remote
                                 10.9.0.5              -       100     0       65999 65299 i
 * >Ec    RD: 10.9.0.5:1 ip-prefix 0.0.0.0/0
                                 -                     -       100     0       65301 ?
 *  ec    RD: 10.9.0.5:1 ip-prefix 0.0.0.0/0
                                 -                     -       100     0       65301 ?
 * >Ec    RD: 10.9.0.5:2 ip-prefix 0.0.0.0/0
                                 -                     -       100     0       65301 ?
 *  ec    RD: 10.9.0.5:2 ip-prefix 0.0.0.0/0
                                 -                     -       100     0       65301 ?
 * >Ec    RD: 10.9.0.5:3 ip-prefix 0.0.0.0/0
                                 -                     -       100     0       65301 ?
 *  ec    RD: 10.9.0.5:3 ip-prefix 0.0.0.0/0
                                 -                     -       100     0       65301 ?
          RD: 10.9.0.6:1 ip-prefix 0.0.0.0/0
                                 10.9.0.5              -       100     0       65200 65299 65301 ?
          RD: 10.9.0.6:1 ip-prefix 0.0.0.0/0
                                 10.9.0.5              -       100     0       65200 65299 65301 ?
          RD: 10.9.0.6:2 ip-prefix 0.0.0.0/0
                                 10.9.0.5              -       100     0       65200 65299 65301 ?
          RD: 10.9.0.6:2 ip-prefix 0.0.0.0/0
                                 10.9.0.5              -       100     0       65200 65299 65301 ?
          RD: 10.9.0.6:3 ip-prefix 0.0.0.0/0
                                 10.9.0.5              -       100     0       65200 65299 65301 ?
          RD: 10.9.0.6:3 ip-prefix 0.0.0.0/0
                                 10.9.0.5              -       100     0       65200 65299 65301 ?
 * >Ec    RD: 10.1.0.5:1 ip-prefix 10.4.0.0/16
                                 10.1.0.5              -       100     0       65999 65198 65100 65101 i
 *  ec    RD: 10.1.0.5:1 ip-prefix 10.4.0.0/16
                                 10.1.0.5              -       100     0       65999 65198 65100 65101 i
          RD: 10.1.0.5:1 ip-prefix 10.4.0.0/16
                                 10.9.0.5              -       100     0       65200 65299 65999 65198 65100 65101 i
          RD: 10.1.0.5:1 ip-prefix 10.4.0.0/16
                                 10.9.0.5              -       100     0       65200 65299 65999 65198 65100 65101 i
 * >Ec    RD: 10.1.0.6:1 ip-prefix 10.4.0.0/16
                                 10.1.0.5              -       100     0       65999 65199 65100 65101 i
 *  ec    RD: 10.1.0.6:1 ip-prefix 10.4.0.0/16
                                 10.1.0.5              -       100     0       65999 65199 65100 65101 i
          RD: 10.1.0.6:1 ip-prefix 10.4.0.0/16
                                 10.9.0.5              -       100     0       65200 65299 65999 65199 65100 65101 i
          RD: 10.1.0.6:1 ip-prefix 10.4.0.0/16
                                 10.9.0.5              -       100     0       65200 65299 65999 65199 65100 65101 i
 * >      RD: 10.9.0.5:1 ip-prefix 10.4.0.0/16
                                 -                     -       -       0       65200 i
          RD: 10.9.0.6:1 ip-prefix 10.4.0.0/16
                                 10.9.0.5              -       100     0       65999 65299 65200 65201 i
          RD: 10.9.0.6:1 ip-prefix 10.4.0.0/16
                                 10.9.0.5              -       100     0       65999 65299 65200 65201 i
 * >Ec    RD: 10.1.0.1:1 ip-prefix 10.4.10.0/24
                                 10.1.0.5              -       100     0       65999 65198 65100 65101 i
 *  ec    RD: 10.1.0.1:1 ip-prefix 10.4.10.0/24
                                 10.1.0.5              -       100     0       65999 65198 65100 65101 i
 * >Ec    RD: 10.1.0.2:1 ip-prefix 10.4.10.0/24
                                 10.1.0.5              -       100     0       65999 65198 65100 65101 i
 *  ec    RD: 10.1.0.2:1 ip-prefix 10.4.10.0/24
                                 10.1.0.5              -       100     0       65999 65198 65100 65101 i
 * >Ec    RD: 10.9.0.1:1 ip-prefix 10.4.10.0/24
                                 10.9.0.1              -       100     0       65200 65201 i
 *  ec    RD: 10.9.0.1:1 ip-prefix 10.4.10.0/24
                                 10.9.0.1              -       100     0       65200 65201 i
 * >Ec    RD: 10.9.0.2:1 ip-prefix 10.4.10.0/24
                                 10.9.0.2              -       100     0       65200 65201 i
 *  ec    RD: 10.9.0.2:1 ip-prefix 10.4.10.0/24
                                 10.9.0.2              -       100     0       65200 65201 i
 * >Ec    RD: 10.9.0.3:1 ip-prefix 10.4.10.0/24
                                 10.9.0.3              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.3:1 ip-prefix 10.4.10.0/24
                                 10.9.0.3              -       100     0       65200 65202 i
 * >Ec    RD: 10.9.0.4:1 ip-prefix 10.4.10.0/24
                                 10.9.0.4              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.4:1 ip-prefix 10.4.10.0/24
                                 10.9.0.4              -       100     0       65200 65202 i
 * >Ec    RD: 10.1.0.5:2 ip-prefix 10.5.0.0/16
                                 10.1.0.5              -       100     0       65999 65198 65100 65101 i
 *  ec    RD: 10.1.0.5:2 ip-prefix 10.5.0.0/16
                                 10.1.0.5              -       100     0       65999 65198 65100 65101 i
          RD: 10.1.0.5:2 ip-prefix 10.5.0.0/16
                                 10.9.0.5              -       100     0       65200 65299 65999 65198 65100 65101 i
          RD: 10.1.0.5:2 ip-prefix 10.5.0.0/16
                                 10.9.0.5              -       100     0       65200 65299 65999 65198 65100 65101 i
 * >Ec    RD: 10.1.0.6:2 ip-prefix 10.5.0.0/16
                                 10.1.0.5              -       100     0       65999 65199 65100 65101 i
 *  ec    RD: 10.1.0.6:2 ip-prefix 10.5.0.0/16
                                 10.1.0.5              -       100     0       65999 65199 65100 65101 i
          RD: 10.1.0.6:2 ip-prefix 10.5.0.0/16
                                 10.9.0.5              -       100     0       65200 65299 65999 65199 65100 65101 i
          RD: 10.1.0.6:2 ip-prefix 10.5.0.0/16
                                 10.9.0.5              -       100     0       65200 65299 65999 65199 65100 65101 i
 * >      RD: 10.9.0.5:2 ip-prefix 10.5.0.0/16
                                 -                     -       -       0       65999 65199 65100 65101 i
          RD: 10.9.0.6:2 ip-prefix 10.5.0.0/16
                                 10.9.0.5              -       100     0       65200 65299 65999 65199 65100 65101 i
          RD: 10.9.0.6:2 ip-prefix 10.5.0.0/16
                                 10.9.0.5              -       100     0       65200 65299 65999 65199 65100 65101 i
 * >Ec    RD: 10.1.0.1:2 ip-prefix 10.5.20.0/24
                                 10.1.0.5              -       100     0       65999 65199 65100 65101 i
 *  ec    RD: 10.1.0.1:2 ip-prefix 10.5.20.0/24
                                 10.1.0.5              -       100     0       65999 65199 65100 65101 i
          RD: 10.1.0.1:2 ip-prefix 10.5.20.0/24
                                 10.9.0.5              -       100     0       65200 65299 65999 65199 65100 65101 i
          RD: 10.1.0.1:2 ip-prefix 10.5.20.0/24
                                 10.9.0.5              -       100     0       65200 65299 65999 65199 65100 65101 i
 * >Ec    RD: 10.1.0.2:2 ip-prefix 10.5.20.0/24
                                 10.1.0.5              -       100     0       65999 65199 65100 65101 i
 *  ec    RD: 10.1.0.2:2 ip-prefix 10.5.20.0/24
                                 10.1.0.5              -       100     0       65999 65199 65100 65101 i
          RD: 10.1.0.2:2 ip-prefix 10.5.20.0/24
                                 10.9.0.5              -       100     0       65200 65299 65999 65199 65100 65101 i
          RD: 10.1.0.2:2 ip-prefix 10.5.20.0/24
                                 10.9.0.5              -       100     0       65200 65299 65999 65199 65100 65101 i
 * >      RD: 10.9.0.5:2 ip-prefix 10.12.0.0/16
                                 -                     -       -       0       65200 i
          RD: 10.9.0.6:2 ip-prefix 10.12.0.0/16
                                 10.9.0.5              -       100     0       65999 65299 65200 i
          RD: 10.9.0.6:2 ip-prefix 10.12.0.0/16
                                 10.9.0.5              -       100     0       65999 65299 65200 i
 * >Ec    RD: 10.9.0.1:2 ip-prefix 10.12.20.0/24
                                 10.9.0.1              -       100     0       65200 65201 i
 *  ec    RD: 10.9.0.1:2 ip-prefix 10.12.20.0/24
                                 10.9.0.1              -       100     0       65200 65201 i
 * >Ec    RD: 10.9.0.2:2 ip-prefix 10.12.20.0/24
                                 10.9.0.2              -       100     0       65200 65201 i
 *  ec    RD: 10.9.0.2:2 ip-prefix 10.12.20.0/24
                                 10.9.0.2              -       100     0       65200 65201 i
 * >Ec    RD: 10.9.0.3:2 ip-prefix 10.12.20.0/24
                                 10.9.0.3              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.3:2 ip-prefix 10.12.20.0/24
                                 10.9.0.3              -       100     0       65200 65202 i
 * >Ec    RD: 10.9.0.4:2 ip-prefix 10.12.20.0/24
                                 10.9.0.4              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.4:2 ip-prefix 10.12.20.0/24
                                 10.9.0.4              -       100     0       65200 65202 i
 * >      RD: 10.9.0.5:3 ip-prefix 10.13.0.0/16
                                 -                     -       -       0       65200 i
          RD: 10.9.0.6:3 ip-prefix 10.13.0.0/16
                                 10.9.0.5              -       100     0       65999 65299 65200 i
          RD: 10.9.0.6:3 ip-prefix 10.13.0.0/16
                                 10.9.0.5              -       100     0       65999 65299 65200 i
 * >Ec    RD: 10.9.0.1:3 ip-prefix 10.13.30.0/24
                                 10.9.0.1              -       100     0       65200 65201 i
 *  ec    RD: 10.9.0.1:3 ip-prefix 10.13.30.0/24
                                 10.9.0.1              -       100     0       65200 65201 i
 * >Ec    RD: 10.9.0.2:3 ip-prefix 10.13.30.0/24
                                 10.9.0.2              -       100     0       65200 65201 i
 *  ec    RD: 10.9.0.2:3 ip-prefix 10.13.30.0/24
                                 10.9.0.2              -       100     0       65200 65201 i
 * >Ec    RD: 10.9.0.3:3 ip-prefix 10.13.30.0/24
                                 10.9.0.3              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.3:3 ip-prefix 10.13.30.0/24
                                 10.9.0.3              -       100     0       65200 65202 i
 * >Ec    RD: 10.9.0.4:3 ip-prefix 10.13.30.0/24
                                 10.9.0.4              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.4:3 ip-prefix 10.13.30.0/24
                                 10.9.0.4              -       100     0       65200 65202 i
 * >      RD: 10.9.0.5:1 ip-prefix 10.14.0.0/16
                                 -                     -       -       0       65200 65201 i
          RD: 10.9.0.6:1 ip-prefix 10.14.0.0/16
                                 10.9.0.5              -       100     0       65999 65299 65200 65201 i
          RD: 10.9.0.6:1 ip-prefix 10.14.0.0/16
                                 10.9.0.5              -       100     0       65999 65299 65200 65201 i
 * >Ec    RD: 10.9.0.1:1 ip-prefix 10.14.14.0/24
                                 10.9.0.1              -       100     0       65200 65201 i
 *  ec    RD: 10.9.0.1:1 ip-prefix 10.14.14.0/24
                                 10.9.0.1              -       100     0       65200 65201 i
 * >Ec    RD: 10.9.0.2:1 ip-prefix 10.14.14.0/24
                                 10.9.0.2              -       100     0       65200 65201 i
 *  ec    RD: 10.9.0.2:1 ip-prefix 10.14.14.0/24
                                 10.9.0.2              -       100     0       65200 65201 i
          RD: 10.9.0.6:1 ip-prefix 0.0.0.0/0 remote
                                 10.9.0.5              -       100     0       65200 65299 65301 ?
          RD: 10.9.0.6:1 ip-prefix 0.0.0.0/0 remote
                                 10.9.0.5              -       100     0       65200 65299 65301 ?
          RD: 10.9.0.6:2 ip-prefix 0.0.0.0/0 remote
                                 10.9.0.5              -       100     0       65200 65299 65301 ?
          RD: 10.9.0.6:2 ip-prefix 0.0.0.0/0 remote
                                 10.9.0.5              -       100     0       65200 65299 65301 ?
          RD: 10.9.0.6:3 ip-prefix 0.0.0.0/0 remote
                                 10.9.0.5              -       100     0       65200 65299 65301 ?
          RD: 10.9.0.6:3 ip-prefix 0.0.0.0/0 remote
                                 10.9.0.5              -       100     0       65200 65299 65301 ?
 * >Ec    RD: 10.1.0.5:1 ip-prefix 10.4.0.0/16 remote
                                 10.1.0.5              -       100     0       65999 65198 65100 65101 i
 *  ec    RD: 10.1.0.5:1 ip-prefix 10.4.0.0/16 remote
                                 10.1.0.5              -       100     0       65999 65198 65100 65101 i
          RD: 10.1.0.5:1 ip-prefix 10.4.0.0/16 remote
                                 10.9.0.5              -       100     0       65200 65299 65999 65198 65100 65101 i
          RD: 10.1.0.5:1 ip-prefix 10.4.0.0/16 remote
                                 10.9.0.5              -       100     0       65200 65299 65999 65198 65100 65101 i
 * >Ec    RD: 10.1.0.6:1 ip-prefix 10.4.0.0/16 remote
                                 10.1.0.5              -       100     0       65999 65199 65100 65101 i
 *  ec    RD: 10.1.0.6:1 ip-prefix 10.4.0.0/16 remote
                                 10.1.0.5              -       100     0       65999 65199 65100 65101 i
          RD: 10.1.0.6:1 ip-prefix 10.4.0.0/16 remote
                                 10.9.0.5              -       100     0       65200 65299 65999 65199 65100 65101 i
          RD: 10.1.0.6:1 ip-prefix 10.4.0.0/16 remote
                                 10.9.0.5              -       100     0       65200 65299 65999 65199 65100 65101 i
 * >      RD: 10.9.0.5:1 ip-prefix 10.4.0.0/16 remote
                                 -                     -       -       0       65200 i
          RD: 10.9.0.6:1 ip-prefix 10.4.0.0/16 remote
                                 10.9.0.5              -       100     0       65999 65299 65200 65201 i
          RD: 10.9.0.6:1 ip-prefix 10.4.0.0/16 remote
                                 10.9.0.5              -       100     0       65999 65299 65200 65201 i
 * >Ec    RD: 10.1.0.1:1 ip-prefix 10.4.10.0/24 remote
                                 10.1.0.5              -       100     0       65999 65198 65100 65101 i
 *  ec    RD: 10.1.0.1:1 ip-prefix 10.4.10.0/24 remote
                                 10.1.0.5              -       100     0       65999 65198 65100 65101 i
 * >Ec    RD: 10.1.0.2:1 ip-prefix 10.4.10.0/24 remote
                                 10.1.0.5              -       100     0       65999 65198 65100 65101 i
 *  ec    RD: 10.1.0.2:1 ip-prefix 10.4.10.0/24 remote
                                 10.1.0.5              -       100     0       65999 65198 65100 65101 i
 * >Ec    RD: 10.9.0.1:1 ip-prefix 10.4.10.0/24 remote
                                 10.9.0.1              -       100     0       65200 65201 i
 *  ec    RD: 10.9.0.1:1 ip-prefix 10.4.10.0/24 remote
                                 10.9.0.1              -       100     0       65200 65201 i
 * >Ec    RD: 10.9.0.2:1 ip-prefix 10.4.10.0/24 remote
                                 10.9.0.2              -       100     0       65200 65201 i
 *  ec    RD: 10.9.0.2:1 ip-prefix 10.4.10.0/24 remote
                                 10.9.0.2              -       100     0       65200 65201 i
 * >Ec    RD: 10.9.0.3:1 ip-prefix 10.4.10.0/24 remote
                                 10.9.0.3              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.3:1 ip-prefix 10.4.10.0/24 remote
                                 10.9.0.3              -       100     0       65200 65202 i
 * >Ec    RD: 10.9.0.4:1 ip-prefix 10.4.10.0/24 remote
                                 10.9.0.4              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.4:1 ip-prefix 10.4.10.0/24 remote
                                 10.9.0.4              -       100     0       65200 65202 i
 * >Ec    RD: 10.1.0.5:2 ip-prefix 10.5.0.0/16 remote
                                 10.1.0.5              -       100     0       65999 65198 65100 65101 i
 *  ec    RD: 10.1.0.5:2 ip-prefix 10.5.0.0/16 remote
                                 10.1.0.5              -       100     0       65999 65198 65100 65101 i
          RD: 10.1.0.5:2 ip-prefix 10.5.0.0/16 remote
                                 10.9.0.5              -       100     0       65200 65299 65999 65198 65100 65101 i
          RD: 10.1.0.5:2 ip-prefix 10.5.0.0/16 remote
                                 10.9.0.5              -       100     0       65200 65299 65999 65198 65100 65101 i
 * >Ec    RD: 10.1.0.6:2 ip-prefix 10.5.0.0/16 remote
                                 10.1.0.5              -       100     0       65999 65199 65100 65101 i
 *  ec    RD: 10.1.0.6:2 ip-prefix 10.5.0.0/16 remote
                                 10.1.0.5              -       100     0       65999 65199 65100 65101 i
          RD: 10.1.0.6:2 ip-prefix 10.5.0.0/16 remote
                                 10.9.0.5              -       100     0       65200 65299 65999 65199 65100 65101 i
          RD: 10.1.0.6:2 ip-prefix 10.5.0.0/16 remote
                                 10.9.0.5              -       100     0       65200 65299 65999 65199 65100 65101 i
 * >      RD: 10.9.0.5:2 ip-prefix 10.5.0.0/16 remote
                                 -                     -       -       0       65999 65199 65100 65101 i
          RD: 10.9.0.6:2 ip-prefix 10.5.0.0/16 remote
                                 10.9.0.5              -       100     0       65200 65299 65999 65199 65100 65101 i
          RD: 10.9.0.6:2 ip-prefix 10.5.0.0/16 remote
                                 10.9.0.5              -       100     0       65200 65299 65999 65199 65100 65101 i
 * >Ec    RD: 10.1.0.1:2 ip-prefix 10.5.20.0/24 remote
                                 10.1.0.5              -       100     0       65999 65199 65100 65101 i
 *  ec    RD: 10.1.0.1:2 ip-prefix 10.5.20.0/24 remote
                                 10.1.0.5              -       100     0       65999 65199 65100 65101 i
          RD: 10.1.0.1:2 ip-prefix 10.5.20.0/24 remote
                                 10.9.0.5              -       100     0       65200 65299 65999 65199 65100 65101 i
          RD: 10.1.0.1:2 ip-prefix 10.5.20.0/24 remote
                                 10.9.0.5              -       100     0       65200 65299 65999 65199 65100 65101 i
 * >Ec    RD: 10.1.0.2:2 ip-prefix 10.5.20.0/24 remote
                                 10.1.0.5              -       100     0       65999 65199 65100 65101 i
 *  ec    RD: 10.1.0.2:2 ip-prefix 10.5.20.0/24 remote
                                 10.1.0.5              -       100     0       65999 65199 65100 65101 i
          RD: 10.1.0.2:2 ip-prefix 10.5.20.0/24 remote
                                 10.9.0.5              -       100     0       65200 65299 65999 65199 65100 65101 i
          RD: 10.1.0.2:2 ip-prefix 10.5.20.0/24 remote
                                 10.9.0.5              -       100     0       65200 65299 65999 65199 65100 65101 i
 * >      RD: 10.9.0.5:2 ip-prefix 10.12.0.0/16 remote
                                 -                     -       -       0       65200 i
          RD: 10.9.0.6:2 ip-prefix 10.12.0.0/16 remote
                                 10.9.0.5              -       100     0       65999 65299 65200 i
          RD: 10.9.0.6:2 ip-prefix 10.12.0.0/16 remote
                                 10.9.0.5              -       100     0       65999 65299 65200 i
 * >Ec    RD: 10.9.0.1:2 ip-prefix 10.12.20.0/24 remote
                                 10.9.0.1              -       100     0       65200 65201 i
 *  ec    RD: 10.9.0.1:2 ip-prefix 10.12.20.0/24 remote
                                 10.9.0.1              -       100     0       65200 65201 i
 * >Ec    RD: 10.9.0.2:2 ip-prefix 10.12.20.0/24 remote
                                 10.9.0.2              -       100     0       65200 65201 i
 *  ec    RD: 10.9.0.2:2 ip-prefix 10.12.20.0/24 remote
                                 10.9.0.2              -       100     0       65200 65201 i
 * >Ec    RD: 10.9.0.3:2 ip-prefix 10.12.20.0/24 remote
                                 10.9.0.3              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.3:2 ip-prefix 10.12.20.0/24 remote
                                 10.9.0.3              -       100     0       65200 65202 i
 * >Ec    RD: 10.9.0.4:2 ip-prefix 10.12.20.0/24 remote
                                 10.9.0.4              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.4:2 ip-prefix 10.12.20.0/24 remote
                                 10.9.0.4              -       100     0       65200 65202 i
 * >      RD: 10.9.0.5:3 ip-prefix 10.13.0.0/16 remote
                                 -                     -       -       0       65200 i
          RD: 10.9.0.6:3 ip-prefix 10.13.0.0/16 remote
                                 10.9.0.5              -       100     0       65999 65299 65200 i
          RD: 10.9.0.6:3 ip-prefix 10.13.0.0/16 remote
                                 10.9.0.5              -       100     0       65999 65299 65200 i
 * >Ec    RD: 10.9.0.1:3 ip-prefix 10.13.30.0/24 remote
                                 10.9.0.1              -       100     0       65200 65201 i
 *  ec    RD: 10.9.0.1:3 ip-prefix 10.13.30.0/24 remote
                                 10.9.0.1              -       100     0       65200 65201 i
 * >Ec    RD: 10.9.0.2:3 ip-prefix 10.13.30.0/24 remote
                                 10.9.0.2              -       100     0       65200 65201 i
 *  ec    RD: 10.9.0.2:3 ip-prefix 10.13.30.0/24 remote
                                 10.9.0.2              -       100     0       65200 65201 i
 * >Ec    RD: 10.9.0.3:3 ip-prefix 10.13.30.0/24 remote
                                 10.9.0.3              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.3:3 ip-prefix 10.13.30.0/24 remote
                                 10.9.0.3              -       100     0       65200 65202 i
 * >Ec    RD: 10.9.0.4:3 ip-prefix 10.13.30.0/24 remote
                                 10.9.0.4              -       100     0       65200 65202 i
 *  ec    RD: 10.9.0.4:3 ip-prefix 10.13.30.0/24 remote
                                 10.9.0.4              -       100     0       65200 65202 i
 * >      RD: 10.9.0.5:1 ip-prefix 10.14.0.0/16 remote
                                 -                     -       -       0       65200 65201 i
          RD: 10.9.0.6:1 ip-prefix 10.14.0.0/16 remote
                                 10.9.0.5              -       100     0       65999 65299 65200 65201 i
          RD: 10.9.0.6:1 ip-prefix 10.14.0.0/16 remote
                                 10.9.0.5              -       100     0       65999 65299 65200 65201 i
 * >Ec    RD: 10.9.0.1:1 ip-prefix 10.14.14.0/24 remote
                                 10.9.0.1              -       100     0       65200 65201 i
 *  ec    RD: 10.9.0.1:1 ip-prefix 10.14.14.0/24 remote
                                 10.9.0.1              -       100     0       65200 65201 i
 * >Ec    RD: 10.9.0.2:1 ip-prefix 10.14.14.0/24 remote
                                 10.9.0.2              -       100     0       65200 65201 i
 *  ec    RD: 10.9.0.2:1 ip-prefix 10.14.14.0/24 remote
                                 10.9.0.2              -       100     0       65200 65201 i
DC2-Border-GW1#show ip route vrf BLUE

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
 B E      0.0.0.0/0 [200/0] via 10.198.104.6, Ethernet3.104
                            via 10.198.104.14, Ethernet6.114

 B E      10.4.10.21/32 [200/0] via VTEP 10.9.0.2 VNI 11001 router-mac 50:00:00:43:40:cf local-interface Vxlan1
                                via VTEP 10.9.0.1 VNI 11001 router-mac 50:00:00:d8:ac:19 local-interface Vxlan1
 B E      10.4.10.0/24 [200/0] via VTEP 10.9.0.3 VNI 11001 router-mac 50:00:00:09:ef:21 local-interface Vxlan1
                               via VTEP 10.9.0.4 VNI 11001 router-mac 50:00:00:d5:e2:ad local-interface Vxlan1
                               via VTEP 10.9.0.2 VNI 11001 router-mac 50:00:00:43:40:cf local-interface Vxlan1
                               via VTEP 10.9.0.1 VNI 11001 router-mac 50:00:00:d8:ac:19 local-interface Vxlan1
 B E      10.14.14.0/24 [200/0] via VTEP 10.9.0.2 VNI 11001 router-mac 50:00:00:43:40:cf local-interface Vxlan1
                                via VTEP 10.9.0.1 VNI 11001 router-mac 50:00:00:d8:ac:19 local-interface Vxlan1
 C        10.198.104.4/30 is directly connected, Ethernet3.104
 C        10.198.104.12/30 is directly connected, Ethernet6.114

DC2-Border-GW1#show ip route vrf RED

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
 B E      0.0.0.0/0 [200/0] via 10.198.105.6, Ethernet3.105
                            via 10.198.105.14, Ethernet6.115

 B E      10.5.20.0/24 [200/0] via VTEP 10.1.0.5 VNI 11002 router-mac 00:00:00:10:10:11 local-interface Vxlan1
 B E      10.12.20.24/32 [200/0] via VTEP 10.9.0.3 VNI 11002 router-mac 50:00:00:09:ef:21 local-interface Vxlan1
                                 via VTEP 10.9.0.4 VNI 11002 router-mac 50:00:00:d5:e2:ad local-interface Vxlan1
 B E      10.12.20.0/24 [200/0] via VTEP 10.9.0.1 VNI 11002 router-mac 50:00:00:d8:ac:19 local-interface Vxlan1
                                via VTEP 10.9.0.2 VNI 11002 router-mac 50:00:00:43:40:cf local-interface Vxlan1
                                via VTEP 10.9.0.3 VNI 11002 router-mac 50:00:00:09:ef:21 local-interface Vxlan1
                                via VTEP 10.9.0.4 VNI 11002 router-mac 50:00:00:d5:e2:ad local-interface Vxlan1
 C        10.198.105.4/30 is directly connected, Ethernet3.105
 C        10.198.105.12/30 is directly connected, Ethernet6.115

DC2-Border-GW1#show ip route vrf GREEN

VRF: GREEN
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
 B E      0.0.0.0/0 [200/0] via 10.198.106.6, Ethernet3.106
                            via 10.198.106.14, Ethernet6.116

 B E      10.13.30.22/32 [200/0] via VTEP 10.9.0.2 VNI 11003 router-mac 50:00:00:43:40:cf local-interface Vxlan1
                                 via VTEP 10.9.0.1 VNI 11003 router-mac 50:00:00:d8:ac:19 local-interface Vxlan1
 B E      10.13.30.23/32 [200/0] via VTEP 10.9.0.3 VNI 11003 router-mac 50:00:00:09:ef:21 local-interface Vxlan1
                                 via VTEP 10.9.0.4 VNI 11003 router-mac 50:00:00:d5:e2:ad local-interface Vxlan1
 B E      10.13.30.0/24 [200/0] via VTEP 10.9.0.2 VNI 11003 router-mac 50:00:00:43:40:cf local-interface Vxlan1
                                via VTEP 10.9.0.3 VNI 11003 router-mac 50:00:00:09:ef:21 local-interface Vxlan1
                                via VTEP 10.9.0.4 VNI 11003 router-mac 50:00:00:d5:e2:ad local-interface Vxlan1
                                via VTEP 10.9.0.1 VNI 11003 router-mac 50:00:00:d8:ac:19 local-interface Vxlan1
 C        10.198.106.4/30 is directly connected, Ethernet3.106
 C        10.198.106.12/30 is directly connected, Ethernet6.116

DC2-Border-GW1#show mac address-table
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
  10    0000.0000.0001    STATIC      Cpu
  10    aabb.cc80.6000    DYNAMIC     Vx1        1       0:01:57 ago
  10    aabb.cc81.6000    DYNAMIC     Vx1        1       0:01:25 ago
  10    aabb.cc81.7000    DYNAMIC     Vx1        1       22:57:34 ago
4092    0000.0000.0001    STATIC      Cpu
4092    5000.0009.ef21    DYNAMIC     Vx1        1       22:57:39 ago
4092    5000.0043.40cf    DYNAMIC     Vx1        1       22:57:40 ago
4092    5000.00d5.e2ad    DYNAMIC     Vx1        1       22:57:38 ago
4092    5000.00d8.ac19    DYNAMIC     Vx1        1       22:57:37 ago
4093    0000.0000.0001    STATIC      Cpu
4093    0000.0010.1011    DYNAMIC     Vx1        1       22:57:42 ago
4093    5000.0009.ef21    DYNAMIC     Vx1        1       22:57:38 ago
4093    5000.0043.40cf    DYNAMIC     Vx1        1       22:57:38 ago
4093    5000.00d5.e2ad    DYNAMIC     Vx1        1       22:57:35 ago
4093    5000.00d8.ac19    DYNAMIC     Vx1        1       22:57:40 ago
4094    0000.0000.0001    STATIC      Cpu
4094    0000.0010.1011    DYNAMIC     Vx1        1       22:57:41 ago
4094    5000.0009.ef21    DYNAMIC     Vx1        1       22:57:39 ago
4094    5000.0043.40cf    DYNAMIC     Vx1        1       22:57:37 ago
4094    5000.00d5.e2ad    DYNAMIC     Vx1        1       22:57:38 ago
4094    5000.00d8.ac19    DYNAMIC     Vx1        1       22:57:37 ago
Total Mac Addresses for this criterion: 21

          Multicast Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       ----        -----
Total Mac Addresses for this criterion: 0
DC2-Border-GW1#show vxlan address-table
          Vxlan Mac Address Table
----------------------------------------------------------------------

VLAN  Mac Address     Type      Prt  VTEP             Moves   Last Move
----  -----------     ----      ---  ----             -----   ---------
  10  aabb.cc80.6000  EVPN      Vx1  10.1.0.5         1       0:02:01 ago
  10  aabb.cc81.6000  EVPN      Vx1  10.9.0.1         1       0:01:30 ago
  10  aabb.cc81.7000  EVPN      Vx1  10.9.0.4         1       22:57:39 ago
4092  5000.0009.ef21  EVPN      Vx1  10.9.0.3         1       22:57:44 ago
4092  5000.0043.40cf  EVPN      Vx1  10.9.0.2         1       22:57:45 ago
4092  5000.00d5.e2ad  EVPN      Vx1  10.9.0.4         1       22:57:43 ago
4092  5000.00d8.ac19  EVPN      Vx1  10.9.0.1         1       22:57:41 ago
4093  0000.0010.1011  EVPN      Vx1  10.1.0.5         1       22:57:46 ago
4093  5000.0009.ef21  EVPN      Vx1  10.9.0.3         1       22:57:43 ago
4093  5000.0043.40cf  EVPN      Vx1  10.9.0.2         1       22:57:43 ago
4093  5000.00d5.e2ad  EVPN      Vx1  10.9.0.4         1       22:57:39 ago
4093  5000.00d8.ac19  EVPN      Vx1  10.9.0.1         1       22:57:45 ago
4094  0000.0010.1011  EVPN      Vx1  10.1.0.5         1       22:57:45 ago
4094  5000.0009.ef21  EVPN      Vx1  10.9.0.3         1       22:57:43 ago
4094  5000.0043.40cf  EVPN      Vx1  10.9.0.2         1       22:57:41 ago
4094  5000.00d5.e2ad  EVPN      Vx1  10.9.0.4         1       22:57:43 ago
4094  5000.00d8.ac19  EVPN      Vx1  10.9.0.1         1       22:57:41 ago
Total Remote Mac Addresses for this criterion: 17
DC2-Border-GW1#show vxlan flood vtep
          VXLAN Flood VTEP Table
--------------------------------------------------------------------------------

VLANS                            Ip Address
-----------------------------   ------------------------------------------------
10                              10.1.0.5        10.9.0.1        10.9.0.2
                                10.9.0.3        10.9.0.4
DC2-Border-GW1#show vxlan vtep
Remote VTEPS for Vxlan1:

VTEP           Tunnel Type(s)
-------------- --------------
10.1.0.5       flood, unicast
10.9.0.1       flood, unicast
10.9.0.2       flood, unicast
10.9.0.3       flood, unicast
10.9.0.4       flood, unicast

Total number of remote VTEPS:  5
DC2-Border-GW1#show vxlan vni
VNI to VLAN Mapping for Vxlan1
VNI         VLAN       Source       Interface       802.1Q Tag
----------- ---------- ------------ --------------- ----------
10010       10         static       Vxlan1          10

VNI to dynamic VLAN Mapping for Vxlan1
VNI         VLAN       VRF         Source
----------- ---------- ----------- ------------
11001       4094       BLUE        evpn
11002       4093       RED         evpn
11003       4092       GREEN       evpn

DC2-Border-GW1#

```
</details>

**6.6 Проверка работы Multi-Domain EVPN VXLAN (Anycast Gateway)**

<details>
<summary> 6.6.1 Получение Type-5 маршрутов между доменами (ЦОДами) для vrf на DC1-Leaf1-1 </summary>

```
DC1-Leaf-1-1#sh ip route vrf BLUE

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
 B E      0.0.0.0/0 [200/0] via VTEP 10.1.0.5 VNI 11001 router-mac 00:00:00:10:10:11 local-interface Vxlan1

 C        10.4.10.0/24 is directly connected, Vlan10
 B E      10.4.0.0/16 [200/0] via VTEP 10.1.0.5 VNI 11001 router-mac 00:00:00:10:10:11 local-interface Vxlan1
 B E      10.14.14.0/24 [200/0] via VTEP 10.1.0.5 VNI 11001 router-mac 00:00:00:10:10:11 local-interface Vxlan1
 B E      10.14.0.0/16 [200/0] via VTEP 10.1.0.5 VNI 11001 router-mac 00:00:00:10:10:11 local-interface Vxlan1

DC1-Leaf-1-1#sh ip route vrf RED

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
 B E      0.0.0.0/0 [200/0] via VTEP 10.1.0.5 VNI 11002 router-mac 00:00:00:10:10:11 local-interface Vxlan1

 C        10.5.20.0/24 is directly connected, Vlan20
 B E      10.12.20.0/24 [200/0] via VTEP 10.1.0.5 VNI 11002 router-mac 00:00:00:10:10:11 local-interface Vxlan1
 B E      10.12.0.0/16 [200/0] via VTEP 10.1.0.5 VNI 11002 router-mac 00:00:00:10:10:11 local-interface Vxlan1

DC1-Leaf-1-1#
DC1-Leaf-1-1#sh bgp evpn route-type ip-prefix 10.12.20.0/24
BGP routing table information for VRF default
Router identifier 10.0.0.1, local AS number 65101
BGP routing table entry for ip-prefix 10.12.20.0/24, Route Distinguisher: 10.9.0.1:2
 Paths: 1 available
  65100 65199 65999 65298 65200 65201
    10.1.0.5 from 10.0.1.0 (10.0.1.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, best
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:00:00:00:10:10:11
      VNI: 11002
BGP routing table entry for ip-prefix 10.12.20.0/24, Route Distinguisher: 10.9.0.2:2
 Paths: 1 available
  65100 65199 65999 65298 65200 65201
    10.1.0.5 from 10.0.1.0 (10.0.1.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, best
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:00:00:00:10:10:11
      VNI: 11002
BGP routing table entry for ip-prefix 10.12.20.0/24, Route Distinguisher: 10.9.0.3:2
 Paths: 1 available
  65100 65199 65999 65298 65200 65202
    10.1.0.5 from 10.0.1.0 (10.0.1.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, best
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:00:00:00:10:10:11
      VNI: 11002
BGP routing table entry for ip-prefix 10.12.20.0/24, Route Distinguisher: 10.9.0.4:2
 Paths: 1 available
  65100 65199 65999 65298 65200 65202
    10.1.0.5 from 10.0.1.0 (10.0.1.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, best
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:00:00:00:10:10:11
      VNI: 11002

```
</details>

<details>
<summary> 6.6.2 Получение Type-5 маршрутов между доменами (ЦОДами) для vrf на DC1-Border-GW1 </summary>

```
DC1-Border-GW1#sh bgp evpn route-type ip-prefix 10.12.20.0/24
BGP routing table information for VRF default
Router identifier 10.0.0.5, local AS number 65198
BGP routing table entry for ip-prefix 10.12.20.0/24, Route Distinguisher: 10.9.0.1:2
 Paths: 3 available
  65999 65298 65200 65201
    10.9.0.5 from - (0.0.0.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:00:00:00:10:10:22
      VNI: 11002
  65999 65298 65200 65201
    10.9.0.5 from - (0.0.0.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:00:00:00:10:10:22
      VNI: 11002
  65100 65199 65999 65298 65200 65201
    10.1.0.5 from 10.0.1.0 (10.0.1.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, invalid, external
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:00:00:00:10:10:11
      VNI: 11002
BGP routing table entry for ip-prefix 10.12.20.0/24, Route Distinguisher: 10.9.0.2:2
 Paths: 3 available
  65999 65298 65200 65201
    10.9.0.5 from - (0.0.0.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:00:00:00:10:10:22
      VNI: 11002
  65999 65298 65200 65201
    10.9.0.5 from - (0.0.0.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:00:00:00:10:10:22
      VNI: 11002
  65100 65199 65999 65298 65200 65201
    10.1.0.5 from 10.0.1.0 (10.0.1.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, invalid, external
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:00:00:00:10:10:11
      VNI: 11002
BGP routing table entry for ip-prefix 10.12.20.0/24, Route Distinguisher: 10.9.0.3:2
 Paths: 3 available
  65999 65298 65200 65202
    10.9.0.5 from - (0.0.0.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:00:00:00:10:10:22
      VNI: 11002
  65999 65298 65200 65202
    10.9.0.5 from - (0.0.0.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:00:00:00:10:10:22
      VNI: 11002
  65100 65199 65999 65298 65200 65202
    10.1.0.5 from 10.0.1.0 (10.0.1.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, invalid, external
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:00:00:00:10:10:11
      VNI: 11002
BGP routing table entry for ip-prefix 10.12.20.0/24, Route Distinguisher: 10.9.0.4:2
 Paths: 3 available
  65999 65298 65200 65202
    10.9.0.5 from - (0.0.0.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:00:00:00:10:10:22
      VNI: 11002
  65999 65298 65200 65202
    10.9.0.5 from - (0.0.0.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:00:00:00:10:10:22
      VNI: 11002
  65100 65199 65999 65298 65200 65202
    10.1.0.5 from 10.0.1.0 (10.0.1.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, invalid, external
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:00:00:00:10:10:11
      VNI: 11002
BGP routing table entry for ip-prefix 10.12.20.0/24 remote, Route Distinguisher: 10.9.0.1:2
 Paths: 3 available
  65999 65298 65200 65201
    10.9.0.5 from 10.91.91.2 (10.91.91.2)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:00:00:00:10:10:22
      VNI: 11002
  65999 65298 65200 65201
    10.9.0.5 from 10.91.91.1 (10.91.91.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:00:00:00:10:10:22
      VNI: 11002
  65100 65199 65999 65298 65200 65201
    10.1.0.5 from - (0.0.0.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, invalid, external
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:00:00:00:10:10:11
      VNI: 11002
BGP routing table entry for ip-prefix 10.12.20.0/24 remote, Route Distinguisher: 10.9.0.2:2
 Paths: 3 available
  65999 65298 65200 65201
    10.9.0.5 from 10.91.91.1 (10.91.91.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:00:00:00:10:10:22
      VNI: 11002
  65999 65298 65200 65201
    10.9.0.5 from 10.91.91.2 (10.91.91.2)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:00:00:00:10:10:22
      VNI: 11002
  65100 65199 65999 65298 65200 65201
    10.1.0.5 from - (0.0.0.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, invalid, external
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:00:00:00:10:10:11
      VNI: 11002
BGP routing table entry for ip-prefix 10.12.20.0/24 remote, Route Distinguisher: 10.9.0.3:2
 Paths: 3 available
  65999 65298 65200 65202
    10.9.0.5 from 10.91.91.1 (10.91.91.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:00:00:00:10:10:22
      VNI: 11002
  65999 65298 65200 65202
    10.9.0.5 from 10.91.91.2 (10.91.91.2)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:00:00:00:10:10:22
      VNI: 11002
  65100 65199 65999 65298 65200 65202
    10.1.0.5 from - (0.0.0.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, invalid, external
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:00:00:00:10:10:11
      VNI: 11002
BGP routing table entry for ip-prefix 10.12.20.0/24 remote, Route Distinguisher: 10.9.0.4:2
 Paths: 3 available
  65999 65298 65200 65202
    10.9.0.5 from 10.91.91.1 (10.91.91.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:00:00:00:10:10:22
      VNI: 11002
  65999 65298 65200 65202
    10.9.0.5 from 10.91.91.2 (10.91.91.2)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:00:00:00:10:10:22
      VNI: 11002
  65100 65199 65999 65298 65200 65202
    10.1.0.5 from - (0.0.0.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, invalid, external
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:00:00:00:10:10:11
      VNI: 11002
DC1-Border-GW1#

```
</details>

<details>
<summary> 6.6.3 Получение Type-5 маршрутов между доменами (ЦОДами) для vrf на DC2-Border-GW1 </summary>

```

DC2-Border-GW1#sh bgp evpn route-type ip-prefix 10.12.20.0/24
BGP routing table information for VRF default
Router identifier 10.8.0.5, local AS number 65298
BGP routing table entry for ip-prefix 10.12.20.0/24, Route Distinguisher: 10.9.0.1:2
 Paths: 2 available
  65200 65201
    10.9.0.1 from 10.8.1.0 (10.8.1.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:d8:ac:19
      VNI: 11002
  65200 65201
    10.9.0.1 from 10.8.2.0 (10.8.2.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:d8:ac:19
      VNI: 11002
BGP routing table entry for ip-prefix 10.12.20.0/24, Route Distinguisher: 10.9.0.2:2
 Paths: 2 available
  65200 65201
    10.9.0.2 from 10.8.1.0 (10.8.1.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:43:40:cf
      VNI: 11002
  65200 65201
    10.9.0.2 from 10.8.2.0 (10.8.2.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:43:40:cf
      VNI: 11002
BGP routing table entry for ip-prefix 10.12.20.0/24, Route Distinguisher: 10.9.0.3:2
 Paths: 2 available
  65200 65202
    10.9.0.3 from 10.8.1.0 (10.8.1.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:09:ef:21
      VNI: 11002
  65200 65202
    10.9.0.3 from 10.8.2.0 (10.8.2.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:09:ef:21
      VNI: 11002
BGP routing table entry for ip-prefix 10.12.20.0/24, Route Distinguisher: 10.9.0.4:2
 Paths: 2 available
  65200 65202
    10.9.0.4 from 10.8.1.0 (10.8.1.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:d5:e2:ad
      VNI: 11002
  65200 65202
    10.9.0.4 from 10.8.2.0 (10.8.2.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:d5:e2:ad
      VNI: 11002
BGP routing table entry for ip-prefix 10.12.20.0/24 remote, Route Distinguisher: 10.9.0.1:2
 Paths: 2 available
  65200 65201
    10.9.0.1 from - (0.0.0.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:d8:ac:19
      VNI: 11002
  65200 65201
    10.9.0.1 from - (0.0.0.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:d8:ac:19
      VNI: 11002
BGP routing table entry for ip-prefix 10.12.20.0/24 remote, Route Distinguisher: 10.9.0.2:2
 Paths: 2 available
  65200 65201
    10.9.0.2 from - (0.0.0.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:43:40:cf
      VNI: 11002
  65200 65201
    10.9.0.2 from - (0.0.0.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:43:40:cf
      VNI: 11002
BGP routing table entry for ip-prefix 10.12.20.0/24 remote, Route Distinguisher: 10.9.0.3:2
 Paths: 2 available
  65200 65202
    10.9.0.3 from - (0.0.0.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:09:ef:21
      VNI: 11002
  65200 65202
    10.9.0.3 from - (0.0.0.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:09:ef:21
      VNI: 11002
BGP routing table entry for ip-prefix 10.12.20.0/24 remote, Route Distinguisher: 10.9.0.4:2
 Paths: 2 available
  65200 65202
    10.9.0.4 from - (0.0.0.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:d5:e2:ad
      VNI: 11002
  65200 65202
    10.9.0.4 from - (0.0.0.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:2:10002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:d5:e2:ad
      VNI: 11002
DC2-Border-GW1#

```
</details>

<details>
<summary> 6.6.4 Получение Type-2 маршрутов между доменами (ЦОДами) для vrf на DC1-Leaf-1-1 </summary>

```
DC1-Leaf-1-1#sh bgp evpn route-type mac-ip 10.4.10.21
BGP routing table information for VRF default
Router identifier 10.0.0.1, local AS number 65101
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.0.0.5:20010 mac-ip aabb.cc81.6000 10.4.10.21
                                 10.1.0.5              -       100     0       65100 65198 65999 65299 65200 65201 i
 * >      RD: 10.0.0.6:20010 mac-ip aabb.cc81.6000 10.4.10.21
                                 10.1.0.5              -       100     0       65100 65199 65999 65299 65200 65201 i
DC1-Leaf-1-1#sh bgp evpn route-type mac-ip 10.4.10.21 detail
BGP routing table information for VRF default
Router identifier 10.0.0.1, local AS number 65101
BGP routing table entry for mac-ip aabb.cc81.6000 10.4.10.21, Route Distinguisher: 10.0.0.5:20010
 Paths: 1 available
  65100 65198 65999 65299 65200 65201
    10.1.0.5 from 10.0.1.0 (10.0.1.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, best
      Extended Community: Route-Target-AS:10:10010 Route-Target-AS4:99999:10 TunnelEncap:tunnelTypeVxlan
      VNI: 10010 ESI: 0000:0000:0000:0000:0000
BGP routing table entry for mac-ip aabb.cc81.6000 10.4.10.21, Route Distinguisher: 10.0.0.6:20010
 Paths: 1 available
  65100 65199 65999 65299 65200 65201
    10.1.0.5 from 10.0.1.0 (10.0.1.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, best
      Extended Community: Route-Target-AS:10:10010 Route-Target-AS4:99999:10 TunnelEncap:tunnelTypeVxlan
      VNI: 10010 ESI: 0000:0000:0000:0000:0000
DC1-Leaf-1-1#

```
</details>

<details>
<summary> 6.6.5 Получение Type-2 маршрутов между доменами (ЦОДами) для vrf на DC1-Border-GW1 </summary>

```
DC1-Border-GW1#sh bgp evpn route-type mac-ip 10.4.10.21
BGP routing table information for VRF default
Router identifier 10.0.0.5, local AS number 65198
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.0.0.5:20010 mac-ip aabb.cc81.6000 10.4.10.21
                                 -                     -       100     0       65999 65299 65200 65201 i
          RD: 10.0.0.6:20010 mac-ip aabb.cc81.6000 10.4.10.21
                                 10.1.0.5              -       100     0       65100 65199 65999 65299 65200 65201 i
 * >Ec    RD: 10.8.0.5:20010 mac-ip aabb.cc81.6000 10.4.10.21 remote
                                 10.9.0.5              -       100     0       65999 65298 65200 65201 i
 *  ec    RD: 10.8.0.5:20010 mac-ip aabb.cc81.6000 10.4.10.21 remote
                                 10.9.0.5              -       100     0       65999 65298 65200 65201 i
 * >Ec    RD: 10.8.0.6:20010 mac-ip aabb.cc81.6000 10.4.10.21 remote
                                 10.9.0.5              -       100     0       65999 65299 65200 65201 i
 *  ec    RD: 10.8.0.6:20010 mac-ip aabb.cc81.6000 10.4.10.21 remote
                                 10.9.0.5              -       100     0       65999 65299 65200 65201 i
DC1-Border-GW1#
DC1-Border-GW1#
DC1-Border-GW1#
DC1-Border-GW1#sh bgp evpn route-type mac-ip 10.4.10.21 detail
BGP routing table information for VRF default
Router identifier 10.0.0.5, local AS number 65198
BGP routing table entry for mac-ip aabb.cc81.6000 10.4.10.21, Route Distinguisher: 10.0.0.5:20010
 Paths: 1 available
  65999 65299 65200 65201
    - from - (0.0.0.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, best
      Extended Community: Route-Target-AS:10:10010 Route-Target-AS4:99999:10 TunnelEncap:tunnelTypeVxlan
      VNI: 10010 ESI: 0000:0000:0000:0000:0000
BGP routing table entry for mac-ip aabb.cc81.6000 10.4.10.21, Route Distinguisher: 10.0.0.6:20010
 Paths: 1 available
  65100 65199 65999 65299 65200 65201
    10.1.0.5 from 10.0.1.0 (10.0.1.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, invalid, external
      Extended Community: Route-Target-AS:10:10010 Route-Target-AS4:99999:10 TunnelEncap:tunnelTypeVxlan
      VNI: 10010 ESI: 0000:0000:0000:0000:0000
BGP routing table entry for mac-ip aabb.cc81.6000 10.4.10.21 remote, Route Distinguisher: 10.8.0.5:20010
 Paths: 2 available
  65999 65298 65200 65201
    10.9.0.5 from 10.91.91.1 (10.91.91.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS4:99999:10 TunnelEncap:tunnelTypeVxlan
      VNI: 10010 ESI: 0000:0000:0000:0000:0000
  65999 65298 65200 65201
    10.9.0.5 from 10.91.91.2 (10.91.91.2)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS4:99999:10 TunnelEncap:tunnelTypeVxlan
      VNI: 10010 ESI: 0000:0000:0000:0000:0000
BGP routing table entry for mac-ip aabb.cc81.6000 10.4.10.21 remote, Route Distinguisher: 10.8.0.6:20010
 Paths: 2 available
  65999 65299 65200 65201
    10.9.0.5 from 10.91.91.2 (10.91.91.2)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS4:99999:10 TunnelEncap:tunnelTypeVxlan
      VNI: 10010 ESI: 0000:0000:0000:0000:0000
  65999 65299 65200 65201
    10.9.0.5 from 10.91.91.1 (10.91.91.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS4:99999:10 TunnelEncap:tunnelTypeVxlan
      VNI: 10010 ESI: 0000:0000:0000:0000:0000
DC1-Border-GW1#

```
</details>

<details>
<summary> 6.6.6 Получение Type-2 маршрутов между доменами (ЦОДами) для vrf на DC2-Border-GW1 </summary>

```
DC2-Border-GW1#sh bgp evpn route-type mac-ip 10.4.10.21
BGP routing table information for VRF default
Router identifier 10.8.0.5, local AS number 65298
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.9.0.1:10010 mac-ip aabb.cc81.6000 10.4.10.21
                                 10.9.0.1              -       100     0       65200 65201 i
 *  ec    RD: 10.9.0.1:10010 mac-ip aabb.cc81.6000 10.4.10.21
                                 10.9.0.1              -       100     0       65200 65201 i
 * >Ec    RD: 10.9.0.2:10010 mac-ip aabb.cc81.6000 10.4.10.21
                                 10.9.0.2              -       100     0       65200 65201 i
 *  ec    RD: 10.9.0.2:10010 mac-ip aabb.cc81.6000 10.4.10.21
                                 10.9.0.2              -       100     0       65200 65201 i
 * >      RD: 10.8.0.5:20010 mac-ip aabb.cc81.6000 10.4.10.21 remote
                                 -                     -       100     0       65200 65201 i
          RD: 10.8.0.6:20010 mac-ip aabb.cc81.6000 10.4.10.21 remote
                                 10.9.0.5              -       100     0       65999 65299 65200 65201 i
          RD: 10.8.0.6:20010 mac-ip aabb.cc81.6000 10.4.10.21 remote
                                 10.9.0.5              -       100     0       65999 65299 65200 65201 i
DC2-Border-GW1#
DC2-Border-GW1#
DC2-Border-GW1#
DC2-Border-GW1#sh bgp evpn route-type mac-ip 10.4.10.21 detail
BGP routing table information for VRF default
Router identifier 10.8.0.5, local AS number 65298
BGP routing table entry for mac-ip aabb.cc81.6000 10.4.10.21, Route Distinguisher: 10.9.0.1:10010
 Paths: 2 available
  65200 65201
    10.9.0.1 from 10.8.2.0 (10.8.2.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:1:10001 Route-Target-AS:10:10010 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:d8:ac:19
      VNI: 10010 L3 VNI: 11001 ESI: 0000:0000:0000:0000:0000
  65200 65201
    10.9.0.1 from 10.8.1.0 (10.8.1.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:1:10001 Route-Target-AS:10:10010 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:d8:ac:19
      VNI: 10010 L3 VNI: 11001 ESI: 0000:0000:0000:0000:0000
BGP routing table entry for mac-ip aabb.cc81.6000 10.4.10.21, Route Distinguisher: 10.9.0.2:10010
 Paths: 2 available
  65200 65201
    10.9.0.2 from 10.8.1.0 (10.8.1.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:1:10001 Route-Target-AS:10:10010 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:43:40:cf
      VNI: 10010 L3 VNI: 11001 ESI: 0000:0000:0000:0000:0000
  65200 65201
    10.9.0.2 from 10.8.2.0 (10.8.2.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:1:10001 Route-Target-AS:10:10010 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:43:40:cf
      VNI: 10010 L3 VNI: 11001 ESI: 0000:0000:0000:0000:0000
BGP routing table entry for mac-ip aabb.cc81.6000 10.4.10.21 remote, Route Distinguisher: 10.8.0.5:20010
 Paths: 1 available
  65200 65201
    - from - (0.0.0.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, best
      Extended Community: Route-Target-AS4:99999:10 TunnelEncap:tunnelTypeVxlan
      VNI: 10010 ESI: 0000:0000:0000:0000:0000
BGP routing table entry for mac-ip aabb.cc81.6000 10.4.10.21 remote, Route Distinguisher: 10.8.0.6:20010
 Paths: 2 available
  65999 65299 65200 65201
    10.9.0.5 from 10.91.91.1 (10.91.91.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, invalid, external
      Extended Community: Route-Target-AS4:99999:10 TunnelEncap:tunnelTypeVxlan
      VNI: 10010 ESI: 0000:0000:0000:0000:0000
  65999 65299 65200 65201
    10.9.0.5 from 10.91.91.2 (10.91.91.2)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, invalid, external
      Extended Community: Route-Target-AS4:99999:10 TunnelEncap:tunnelTypeVxlan
      VNI: 10010 ESI: 0000:0000:0000:0000:0000
DC2-Border-GW1#

```
</details>


<details>
<summary> 6.6.7 Проверка vxlan flood доменов </summary>

```
DC1-Leaf-1-1#show vxlan flood vtep
          VXLAN Flood VTEP Table
--------------------------------------------------------------------------------

VLANS                            Ip Address
-----------------------------   ------------------------------------------------
10                              10.1.0.5

DC1-Border-GW1#show vxlan flood vtep
          VXLAN Flood VTEP Table
--------------------------------------------------------------------------------

VLANS                            Ip Address
-----------------------------   ------------------------------------------------
10                              10.1.0.1        10.1.0.2        10.9.0.5
DC1-Border-GW1#

DC2-Border-GW1#show vxlan flood vtep
          VXLAN Flood VTEP Table
--------------------------------------------------------------------------------

VLANS                            Ip Address
-----------------------------   ------------------------------------------------
10                              10.1.0.5        10.9.0.1        10.9.0.2
                                10.9.0.3        10.9.0.4
  
DC2-Leaf-1-1#show vxlan flood vtep
          VXLAN Flood VTEP Table
--------------------------------------------------------------------------------

VLANS                            Ip Address
-----------------------------   ------------------------------------------------
10                              10.9.0.3        10.9.0.4        10.9.0.5
20,30                           10.9.0.3        10.9.0.4
DC2-Leaf-1-1#

```
</details>

### 7. Итоги:
Из выводов по результатам проверок видно, что поставленные задачи выполнены, цели проекта достигнуты:
- между фабриками распространяются Type-2 и Type-5 маршруты для продуктивного vrf, что позволит осуществлять миграцию ВМ и обеспечивать резервирование между ЦОД;
- для остальных vrf обеспечена прямая связность между ЦОДами благодаря маршрутам Type-5 распространяющихся между фабриками через DCI;
- DCI стык зарезервирован, Border-GW имеют один Anycast IP;
- Flood домены EVPN VXLAN разделены между собой,  Border-GW подменяют nexthop на себя.








