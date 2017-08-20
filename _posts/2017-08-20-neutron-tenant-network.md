---
layout: post 
author: shalou
title:  "neutron中的tenant network" 
category: openstack
tag: [openstack,neutron]
---


tenant network，即租户网络，是提供给租户的网络，用于租户间隔离，租户网络目前有四种类型，即local、vlan、vxlan和gre。其实也就是neutron中的四种type drivers。下面将依次减少四种网络类型

## 1. local network
local network，从其名字就可以看出，它只能用于部署一个单节点的集群，用来做测试。节点上的同一个network下的虚拟机可以相互通信。

配置ml2时，type_drivers需要包括local，同时tenant_network_types为local

<!-- more -->


```golang
//ml2_conf.ini
[ml2]
type_drivers = flat,vlan,vxlan,local,gre
tenant_network_types = local
```

其网络构造图如下所示

![local network](/images/neutron-tenant-network/local-network.jpg)

其中br-int是一个综合性的ovs网桥，所有虚拟机、或dhcp服务流量最终都要接入该网桥

其中设备A是一个tap设备，它的一端接入KVM虚拟内部，另一端则接入qbr-xxx这个linux网桥设备，B和C是一对veth设备端，用于连接qbr-xxx网桥和br-int网桥。

为什么会存在qbr-xxx这个linux bridge呢？因为openstack的安全组(security group)是通过iptables实现的，iptables不能作用于ovs，而需要作用于qbr-xxx这个网桥


黄色的qdhcp-zzz其实是一个network namespace，里面运行了一个dhcp服务，用于提供dhcp服务，其中X和Y设备虽然命名为tap-zzz，但他们其实是一对veth设备对

C、H、Y这些接入br-int的设备，都会打上一个local vlan的标记，主要用于分布于同一台机上的不同租户间的虚拟的隔离

local network只适合测试，不能用于实际生产中


## 2. vlan network

vlan network比较适合传统用户，vlan一共有4096个，因此不适合来构建公有云。在vlan模式，虚拟机分配是真实的IP，OpenStack集群外可以通过虚拟机的IP直接访问虚拟机，而不需要借助floating ip对外暴露服务

ml2配置vlan模式

```golang
//ml2_conf.ini 
[ml2] 
type_drivers = flat,vlan,vxlan,local,gre
tenant_network_types = vlan

[ml2_type_vlan]
network_vlan_ranges = physnet1:1000:2999

//openvswitch_agent.ini 
bridge_mappings = physnet1:br-eth0
```

![vlan network](/images/neutron-tenant-network/vlan-network.jpg)

相比local network，多出了一个br-eth0这个ovs网桥，这个br-eth0是需要手动创建的，同时要把eth0这张物理网卡添加到br-eth0，当然你也可以添加eth1、eth2，相应的ovs网桥取名为br-eth1、br-eth2，这里的命名其实是随意的，只是为了好识别

M、N两个设备是openvswitch中的patch设备，用于连接两个ovs网桥

C、H那里还是会打上一个local vlan，当流量到了br-eth0时，br-eth0内部有流表，当local vlan转换为真实的vlan tag，然后设置到物理交换机上去

```golang
 cookie=0x812416f0849b631a, duration=3243.293s, table=0, n_packets=2, n_bytes=140, idle_age=3235, priority=4,in_port=3,dl_vlan=3 actions=mod_vlan_vid:1073,NORMAL
```

外界的流量进入br-eth0，然后进入br-int，br-int内部有流表将vlan tag转换为local vlan

```golang
 cookie=0x812416f0849b631a, duration=3371.840s, table=0, n_packets=0, n_bytes=0, idle_age=3371, priority=3,in_port=12,dl_vlan=1073 actions=mod_vlan_vid:3,NORMAL
```

注意这里的eth0需要接入的交换机接口为trunk，这种情况下，一般会把网关配置在交换机上，这样才能直接用内网IP与外界通信

在这种情况下，同一台物理机上的同一个network的虚拟机通过br-int就可以通信，不同物理上同一个network下的虚拟机流量要经过br-int、br-eth0到物理交换机，再进入另一台物理机的br-eth0、br-int、

而虚拟机访问外界、或外界访问虚拟机，直接通过物理交换机就出去了

## 3. vxlan network
vxlan比较适合于公有云，但vxlan不能直接暴露服务，因为它的IP在集群外部看来都是不存在的，因此需要借助l3-agent，添加floating ip

ml2配置vxlan模式

```golang
//ml2_conf.ini 
[ml2] 
type_drivers = flat,vlan,vxlan,local,gre
tenant_network_types = vxlan

[ml2_type_flat]
flat_networks = physnet1

[ml2_type_vxlan]
vni_ranges = 1:1000

//openvswitch_agent.ini 
[ovs]
local_ip = X.X.X.X
bridge_mappings = physnet1:br-ext

[agent]
tunnel_types = vxlan
```

![vxlan network](/images/neutron-tenant-network/vxlan-network.jpg)

在vxlan模式下，多了一个l3组件，他会创建虚拟路由器，也就是图中的qrouter，本质上这是一个network namespace，它一般至少有两张网卡，也就是图中的a和c设备，a设备代表内部某个tenant network的网关，而c设备代表一个外网设备。

在vxlan模式下，多出了一个br-tun设备，br-tun设备是一个ovs网桥，里面的有流表将local vlan装换为vxlan id。br-tun上需要与各个节点(包括计算节点和网络节点)建立点对点的连接，也就是借助K和L设备，他们命名为vxlan-ID。

```golang
NXST_FLOW reply (xid=0x4):
 cookie=0x0, duration=329.194s, table=0, n_packets=31, n_bytes=2906, idle_age=29, priority=1,in_port=1 actions=resubmit(,2)
 cookie=0x0, duration=325.847s, table=0, n_packets=14, n_bytes=1591, idle_age=33, priority=1,in_port=2 actions=resubmit(,4)
 cookie=0x0, duration=328.954s, table=0, n_packets=6, n_bytes=480, idle_age=321, priority=0 actions=drop
 cookie=0x0, duration=328.712s, table=2, n_packets=9, n_bytes=694, idle_age=33, priority=0,dl_dst=00:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,20)
 cookie=0x0, duration=328.465s, table=2, n_packets=22, n_bytes=2212, idle_age=29, priority=0,dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,22)
 cookie=0x0, duration=328.223s, table=3, n_packets=0, n_bytes=0, idle_age=328, priority=0 actions=drop
 cookie=0x0, duration=50.703s, table=4, n_packets=12, n_bytes=1451, idle_age=33, priority=1,tun_id=0x3e9 actions=mod_vlan_vid:1,resubmit(,10)
 cookie=0x0, duration=327.979s, table=4, n_packets=2, n_bytes=140, idle_age=94, priority=0 actions=drop
 cookie=0x0, duration=327.742s, table=10, n_packets=12, n_bytes=1451, idle_age=33, priority=1 actions=learn(table=20,hard_timeout=300,priority=1,NXM_OF_VLAN_TCI[0..11],NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:0->NXM_OF_VLAN_TCI[],load:NXM_NX_TUN_ID[]->NXM_NX_TUN_ID[],output:NXM_OF_IN_PORT[]),output:1
 cookie=0x0, duration=38.551s, table=20, n_packets=9, n_bytes=694, hard_timeout=300, idle_age=33, hard_age=33, priority=1,vlan_tci=0x0001/0x0fff,dl_dst=fa:16:3e:83:95:fa actions=load:0->NXM_OF_VLAN_TCI[],load:0x3e9->NXM_NX_TUN_ID[],output:2
 cookie=0x0, duration=327.504s, table=20, n_packets=0, n_bytes=0, idle_age=327, priority=0 actions=resubmit(,22)
 cookie=0x0, duration=50.94s, table=22, n_packets=11, n_bytes=1334, idle_age=29, dl_vlan=1 actions=strip_vlan,set_tunnel:0x3e9,output:2
 cookie=0x0, duration=327.261s, table=22, n_packets=10, n_bytes=808, idle_age=51, priority=0 actions=drop
```

假如不同的Network需要通信，一般要通过l3-agent创建一个路由器，然后用路由器连接两个network

给虚拟机绑定floating ip，虚拟机才能对外提供服务，其网络流量如下

```golang
虚机eth0 -> 计算节点br-int -> 计算节点br-tun -> 网络节点br-tun -> 网络节点br-int -> qrouter -> 网络节点br-ext -> public
```

## 4. gre network
gre network其实和vxlan基本一样，只是将底层的隧道一些换成了gre，网络拓扑图和vxlan完全一致，br-tun中的流表将local vlan换成gre id

```golang
//ml2_conf.ini 
[ml2] 
type_drivers = flat,vlan,vxlan,local,gre
tenant_network_types = gre

[ml2_type_flat]
flat_networks = physnet1

[ml2_type_gre]
tunnel_id_ranges = 1:1000

//openvswitch_agent.ini 
[ovs]
local_ip = X.X.X.X
bridge_mappings = physnet1:br-ext

[agent]
tunnel_types = gre
```

![gre network](/images/neutron-tenant-network/gre-network.jpg)
