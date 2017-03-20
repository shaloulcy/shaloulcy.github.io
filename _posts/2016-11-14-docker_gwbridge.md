---
layout: post 
author: shalou
title:  "docker网络自定义"
category: 容器技术
tag: [docker,docker\_gwbridge]
---


在docker v1.12版本中引入了swarm mode，同时出现了docker\_gwbridge这个新的网桥，这个网桥的作用其实就是提供docker容器的对外访问的，因为在swarm mode下，假如容器使用了overlay网络，容器是不可以访问网络的，需要再给容器添加一张网卡，这张网卡就桥接在docker\_gwbridge上，你去容器里面会看到，容器有两张网卡，这个过程是自动的。

<!-- more -->

对于docker0,我们可以利用--bip制定docker0的子网地址，官方没提供如何自定义dokcer\_gwbridge的网段的方法，那么如何自定义dokcer\_gwbridge网段了?

假如你执行了docker swarm init. docker\_gwbridge会取一个默认的网段(172.18.0.0)，这时又该如何删除了？

```go
//让节点离开swarm mode
docker swarm leave --force

docker network disconnect docker_gwbridge gateway_ingress-sbox -f //需要加-f，否则会说找不到这个容器

docker network  rm docker_gwbridge

docker network create --subnet 192.168.2.0/24 --opt com.docker.network.bridge.name=docker_gwbridge --opt com.docker.network.bridge.enable_icc=false --opt com.docker.network.bridge.enable_ip_masquerade=true --opt com.docker.network.driver.mtu=1450 docker_gwbridge

docker swarm init
```

bridge网络支持以下自定义参数


|Option|Equivalent|Description|
|:-:|:-:|:-:|
|com.docker.network.bridge.name|-|bridge name to be used when creating the Linux bridge|
|com.docker.network.bridge.enable\_ip\_masquerade|--ip-masq|Enable IP masquerading|
|com.docker.network.bridge.enable_icc|--icc|Enable or Disable Inter Container Connectivity|
|com.docker.network.bridge.host\_binding\_ipv4|--ip|Default IP when binding container ports|
|com.docker.network.driver.mtu|--mtu|Set the containers network MTU|


