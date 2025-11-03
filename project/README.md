## Проектная работа
Проектирование резервного ЦОД с использованием технологий VxLAN EVPN
***
### Цель:
- Организовать резервирование сервисов, находящихся в продуктиве (vrf BLUE), между двумя ЦОД на уровне L2 и L3.
- Организовать взаимодействие остальных сервисов между двумя ЦОД на уровне L3 в пределах своих vrf напрямую, без внешнего маршрутизатора или МСЭ 
#### Задачи:
1. Объединить два ЦОД находящихся каждый в своем домене EVPN VXLAN через L3.
2. Организовать L2 и L3 связность между EVPN доменами разных ЦОД 
3. Обеспечить отказоустойчивость и масштабируемость соединения между ЦОД
4. Обеспечить отказоустойчивость на уровне доступа 
***
## Результаты выполнения:
---
В качестве решения организации интерконнекта между ЦОД используется Multi-Domain EVPN VXLAN (Anycast Gateway), с резервированием пограничных шлюзов (Border GW)
Отказоустойчивость на уровне доступа реализуется с помощью пары MLAG.

- Underlay в обоих ЦОД построен на eBGP
- DC1-Spine-1 и DC1-Spine-2 - находятся в AS65100;
- DC1-Leaf-1-1, DC1-Leaf-1-2 - находятся в AS65101;
- DC1-Leaf-2-1, DC1-Leaf-2-2 - находятся в AS65102;
- DC1-Border-GW1 и DC1-Border-GW2 - находятся в AS65198, AS65199 соответственно.
- DC1-Host-1 и DC1-Host-3 находятся в одном vrf BLUE, подсети 10.4.10.0/24, агрегированный префикс 10.4.0.0/16
- DC1-Host-2 и DC1-Host-4 находятся в одном vrf RED, подсеть 10.5.20.0/24, агрегированный префикс 10.5.0.0/16
- DC2-Spine-1 и DC2-Spine-2 - находятся в AS65200;
- DC2-Leaf-1-1, DC2-Leaf-1-2 - находятся в AS65201;
- DC2-Leaf-2-1, DC2-Leaf-2-2 - находятся в AS65202;
- DC2-Border-GW1 и DC2-Border-GW2 - находятся в AS65298, AS65299 соответственно.
- DC2-Host-1 - находится в vrf BLUE, подсети 10.4.10.0/24, агрегированный префикс 10.4.0.0/16. Также в vrf BLUE есть подсеть 10.14.14.0/24 (агрегированный префикс 10.12.0.0/16)
- DC2-Host-2 и DC2-Host-3 находятся в одном vrf GREEN, подсети 10.13.30.0/24, агрегированный префикс 10.13.0.0/16
- D2-Host-4 находятся vrf RED, подсеть 10.12.20.0/24, агрегированный префикс 10.12.0.0/16
- Route Leaking и выход во внешние каналы осуществляется на граничных маршрутизаторах DC1-Border-R1 и DC2-Border-R1, которые находятся в AS65301

### 1. Схема сети:
   
![](https://github.com/egorvshch/DC-networks-design/blob/main/project/images/schema_dc_net.jpg)
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

#### 3.1 Настройка DC1-Spine-1

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

#### 3.2 Настройка DC1-Spine-2

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

#### 3.3 Настройка DC1-Leaf-1-1

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

#### 3.4 Настройка DC1-Leaf-1-2

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

#### 3.5 Настройка DC1-Leaf-2-1

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

#### 3.6 Настройка DC1-Leaf-2-2

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

#### 3.7 Настройка DC1-Border-GW1

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

#### 3.8 Настройка DC1-Border-GW2

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

#### 3.9 Настройка DC1-Border-R1

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

#### 3.10 Настройка DC1-Host-1, DC1-Host-2, DC1-Host-3 и DC1-Host-4
конфигурация хостов указана по сылками ниже:

[Полная конфигурация DC1-Host-1](https://github.com/egorvshch/DC-networks-design/blob/main/project/configs/DC1-Host-1)

[Полная конфигурация DC1-Host-2](https://github.com/egorvshch/DC-networks-design/blob/main/project/configs/DC1-Host-2)

[Полная конфигурация DC1-Host-3](https://github.com/egorvshch/DC-networks-design/blob/main/project/configs/DC1-Host-3)

[Полная конфигурация DC1-Host-4](https://github.com/egorvshch/DC-networks-design/blob/main/project/configs/DC1-Host-4)

### 4. Настройка оборудрования DC2:

#### 4.1 Настройка DC2-Spine-1

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

#### 4.2 Настройка DC2-Spine-2

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

#### 4.3 Настройка DC2-Leaf-1-1

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

#### 4.4 Настройка DC2-Leaf-1-2

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

#### 4.5 Настройка DC2-Leaf-2-1

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

#### 4.6 Настройка DC2-Leaf-2-2

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

#### 4.7 Настройка DC2-Border-GW1

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

#### 4.8 Настройка DC2-Border-GW2

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

#### 4.9 Настройка DC2-Border-R1

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

#### 4.10 Настройка DC2-Host-1, DC2-Host-2, DC2-Host-3 и DC2-Host-4
конфигурация хостов указана по сылками ниже:

[Полная конфигурация DC2-Host-1](https://github.com/egorvshch/DC-networks-design/blob/main/project/configs/DC2-Host-1)

[Полная конфигурация DC2-Host-2](https://github.com/egorvshch/DC-networks-design/blob/main/project/configs/DC2-Host-2)

[Полная конфигурация DC2-Host-3](https://github.com/egorvshch/DC-networks-design/blob/main/project/configs/DC2-Host-3)

[Полная конфигурация DC2-Host-4](https://github.com/egorvshch/DC-networks-design/blob/main/project/configs/DC2-Host-4)

### 5 Настройка обрудования DCI
#### 5.1 Настройка DCI-Router-1:

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

#### 5.2 Настройка DCI-Router-2:
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



