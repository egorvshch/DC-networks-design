hostname Spine-1 
interface Lo0
ip address 10.0.1.0/32 
no shutdow
interface Lo1 
ip address 10.1.1.0/32 
no shutdow
interface Eth1 
no switchport
ip address 10.2.1.0/31 
description to Leaf-1
no shutdow
interface Eth2 
no switchport
ip address 10.2.1.2/31 
description to Leaf-2
no shutdow
interface Eth3 
no switchport
ip address 10.2.1.4/31 
description to Leaf-3
no shutdow

hostname Spine-2 
interface Lo0 
ip address 10.0.2.0/32 
no shutdow
interface Lo1 
ip address 10.1.2.0/32 
no shutdow
interface Eth1 
no switchport
ip address 10.2.2.0/31
description to Leaf-1
no shutdow
interface Eth2 
no switchport
ip address 10.2.2.2/31
description to Leaf-2
no shutdow
interface Eth3 
no switchport
ip address 10.2.2.4/31
description to Leaf-3
no shutdow

hostname Leaf-1 
interface Lo0 
ip address 10.0.0.1/32 
no shutdow
interface Lo1 
ip address 10.1.0.1/32 
no shutdow
interface Eth1 
no switchport
ip address 10.2.1.1/31
description to Spine-1
no shutdow
interface Eth2 
no switchport
ip address 10.2.2.1/31 
description to Spine-2
no shutdow
interface Eth3 
no switchport
ip address 10.4.1.1/24 
description to Server-1
no shutdow

hostname Leaf-2 
interface Lo0
ip address 10.0.0.2/32 
no shutdow
interface Lo1 
ip address 10.1.0.2/32 
no shutdow
interface Eth1 
no switchport
ip address 10.2.1.3/31
description to Spine-1
no shutdow
interface Eth2 
no switchport
ip address 10.2.2.3/31
description to Spine-2
no shutdow
interface Eth3 
no switchport
ip address 10.4.2.1/24
description to Server-2
no shutdow

hostname Leaf-3 
interface Lo0 
ip address 10.0.0.3/32 
no shutdow
interface Lo1 
ip address 10.1.0.3/32 
no shutdow
interface Eth1 
no switchport
ip address 10.2.1.5/31 
description to Spine-1
no shutdow
interface Eth2 
no switchport
ip address 10.2.2.5/31
description to Spine-2
no shutdow
interface Eth3 
no switchport
ip address 10.4.3.1/24 
description to Server-3
no shutdow
interface Eth4 
no switchport
ip address 10.4.4.1/24
description to Server-4
no shutdow

set pcname Server-1
ip 10.4.1.2/24 10.4.1.1
set pcname Server-2
ip 10.4.2.2/24 10.4.2.1
set pcname Server-3
ip 10.4.3.2/24 10.4.3.1
set pcname Server-4
ip 10.4.4.2/24 10.4.4.1