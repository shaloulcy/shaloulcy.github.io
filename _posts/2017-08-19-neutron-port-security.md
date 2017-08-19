---
layout: post 
author: shalou
title:  "neutron中port的安全性" 
category: openstack
tag: [openstack, neutron]
---

在neutron中有四个概念，即路由器(router)、网络(network)、子网(subnet)和Port:

* router用来连接两个不同的subnet。
* network用来隔离的，一个network根据底层的隔离技术，代表一个vlan id、vxlan id或着gre id，一个network包含多个subnet
* subnet包含的是一段真实的IP段(CIDR)，用来给虚拟机分配IP，一个subnet包含多个port
* port是真实分配给虚拟机的网络资源，一个Port包含一个IP和MAC地址

port即代表一张网卡，插入虚拟机，为虚拟机提供网络服务。准确来说port代表虚拟机交换机上(openvswitch)一个网口，允许虚拟机的网络接入。

那在Neutron中有哪些机制保证Port的安全性

<!-- more -->

## 1. network间的天然隔离
network间利用了vlan、vxlan或gre进行隔离，默认情况下，不同network的port是相互隔离的，假如不同network间的Port想要通信，那么必须添加一个虚拟路由器(router)

## 2. 安全组
安全组，即security-group，通过可以控制port可以跟哪些网络的哪些端口通信，底层是通过iptables实现的

```golang
//创建一个安全组
neutron security-group-show create default

//创建安全组规则，其中424b6f1a-7c0c-4b48-bd23-52bca38af549为安全组的uuid，这条规则表示向10.37.129.0/24这个网段开通22号端口，也就是只有IP地址属于10.37.129.0/24才能访问ssh服务
neutron security-group-rule-create --ethertype IPv4 --protocol tcp --port-range-min 22 --port-range-max 22 --direction ingress --remote-ip-prefix 10.37.129.0/24 424b6f1a-7c0c-4b48-bd23-52bca38af549

//将安全组绑定在f63f3c20-c3b9-480e-b119-f62fa6aecd75这个port上
neutron port-update f63f3c20-c3b9-480e-b119-f62fa6aecd75 --security-group 424b6f1a-7c0c-4b48-bd23-52bca38af549
```

安全组很强大，可以定义ingress和exgress两种流量，可以控制到具体的协议、CIDR和端口范围

## 3. prevent\_arp\_spoofing
prevent\_arp\_spoofing是neutron提供的一种机制，可以防止IP地址和Mac地址咋骗，也就是你不能随意更改分配的IP地址和MAC地址，假如修改了neutron会直接将你的package丢弃，这样就防止用户随意更改网络配置。

neutron底层通过两种方法实现这一机制

### 3.1 iptables机制
neutron会在iptables中写入规则，这事情其实是由安全组模块完成的。它会在iptables中写入如下一条规则

```golang
Chain neutron-openvswi-s26be326e-5 (1 references)
 pkts bytes target     prot opt in     out     source               destination         
  889 72465 RETURN     all  --  *      *       172.17.138.9         0.0.0.0/0            MAC FA:16:3E:78:40:26 /* Allow traffic from defined IP/MAC pairs. */
    0     0 DROP       all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* Drop traffic without an IP/MAC allow rule. */
```

也就是说只有IP地址为172.17.138.9，MAC地址为FA:16:3E:78:40:26才允许通过，其他包则drop掉

### 3.2 flow机制
neutron会在br-int这个ovs网络内部刷入如下两条流表

```
cookie=0x806257ea03ddd617, duration=933.993s, table=24, n_packets=5, n_bytes=210, idle_age=575, priority=2,arp,in_port=8,arp_spa=172.17.138.9 actions=resubmit(,25)
cookie=0x806257ea03ddd617, duration=934.037s, table=25, n_packets=74, n_bytes=9693, idle_age=575, priority=2,in_port=8,dl_src=fa:16:3e:78:40:26 actions=NORMAL
```
只有IP地址为172.17.138.9，mac地址为fa:16:3e:78:40:26才能通过流表

### 3.3 如何关闭
有些业务场景下，我们可能需要关闭prevent\_arp\_spoofing，比如高可用场景下，我们往往需要一个VIP，这个VIP在两台虚拟机之间移动，显然VIP的package是无法通过iptables和流表的，因为我们需要关闭该机制，OpenStack提供了两种方法关闭该机制

#### 3.3.1 allowed\-address\-pairs
通过更新port，让port允许通过某些特殊的IP、MAC地址对

```golang
neutron port-update f63f3c20-c3b9-480e-b119-f62fa6aecd75 --allowed-address-pairs type=dict list=true ip_address=172.17.138.12,mac_address=fa:16:3e:7b:63:cf
```

即允许ip地址为172.17.138.12、Mac地址为fa:16:3e:7b:63:cf通过f63f3c20-c3b9-480e-b119-f62fa6aecd75这个Port，执行上述命令后，neutron会将它刷入iptables和flow里面

显然这种方法只适合地址对比较少的情况

#### 3.3.1 彻底关闭
neutorn给我们提供了彻底关闭某个port安全机制的方法

```golang
//首先删除port绑定的所有的安全组规则
neutron port-update f63f3c20-c3b9-480e-b119-f62fa6aecd75 --no-security-groups

//删除arp spoofing机制
neutron port-update f63f3c20-c3b9-480e-b119-f62fa6aecd75 --port_security_enabled=false
```

这种方法可以彻底arp spoofing，但安全组规则也用不了
