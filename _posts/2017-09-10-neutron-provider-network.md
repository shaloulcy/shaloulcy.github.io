---
layout: post 
author: shalou
title:  "neutron中的provider network" 
category: openstack
tag: [openstack,neutron]
---

neutron中的provider network，是管理员用来创建与外部网络相关联的网络，通过provider network，虚拟机的IP直接就是外界真实的IP，方便与传统网络进行对接，provider network一般是多个租户共享的，provider network的网络类型包括flat、vlan、vxlan、gre和local，但vxlan、gre、local情况下，无法映射真实的物理网络，因此不用，真正可以映射到真实网路的就是flat和vlan，下面将进行介绍

## 1. flat provider network
flat network，即扁平网络，也就是划分出一个单独的网络，这个网络属于某个vlan，从该网络获取IP的虚拟机都属于同一个网络


<!-- more -->

ml2配置flat模式

```golang
//ml2_conf.ini 
[ml2_type_flat]
flat_networks = physnet1

//openvswitch_agent.ini 
bridge_mappings = physnet1:br-eth0


创建命令
neutron net-create --provider:network_type flat --provider:physical_network physnet1 --shared public

neutron subnet-create --name public-subnet --gateway 10.211.55.254  --enable-dhcp --allocation-pool start=10.211.55.100,end=10.211.55.200 public 10.211.55.0/24
```

上述命令将创建一个网络属于10.211.55.0/24网络，IP将从10.211.55.100到10.211.55.200。其物理架构如


![flat-provider-network.jpg](/images/neutron-provider-network/flat-provider-network.jpg)

其实架构图和tenant network中的vlan模式很像，几乎一模一样，主要有一下区别

* eth0这种网卡，在flat模式下，与交换机连接的接口是access模式，而vlan连接的接口是trunk模式
* br-eth0这个ovs网桥的流表不一样

br-eth0流表如下

```golang
cookie=0x97126ea9d8ef97a3, duration=8728.047s, table=0, n_packets=6, n_bytes=468, idle_age=5343, priority=4,in_port=3,dl_vlan=6 actions=strip_vlan,NORMAL
```

仅仅是将local vlan头去除，然后就直接转发出去，这是因为eth0这张网卡已经插在一个access接口上，交换机负责给数据包打上或去除vlan tag

可以同时创建多个flat网络，因为一个flat网络必须关联一个物理网络，所以多个flat网络，就要求有多个物理网络，也就要求有多张网络，配置如下所示

```golang 
//ml2_conf.ini 
[ml2_type_flat]
flat_networks = physnet1,physnet2

//openvswitch_agent.ini 
bridge_mappings = physnet1:br-eth0, physnet2:br-eth1
```

## 2. vlan provider network

vlan provider network其实和vlan tenant network一样，唯一的区别就是vlan provider network是租户共享的，只有管理员能够创建，而vlan tennat network是单独属于某个租户的。

因此其配置，和vlan tenant network一样，只是创建命令不一样


```golang
创建命令
neutron net-create --provider:network_type vlan --provider:physical_network physnet1 --provider:segmentation_id 1072 --shared vlan1072

neutron subnet-create --name public-subnet --gateway 192.168.1.1  --enable-dhcp vlan1072 192.168.1.0/24
```

其架构图

![vlan-provider-network.jpg](/images/neutron-provider-network/vlan-provider-network.jpg)

vlan provider network相对于flat provider network的优势就是，vlan只需要一张网卡，就可以接入多个物理网络，因为它接的交换机口是trunk模式，数据包是由br-eth0流表打上vlan tag再转发到交换机的。相应的vlan provider network需要对交换机做一些额外配置
