---
layout: post 
title:  "自定义docker_gwbridge网段"
categories: docker,swarm mode,docker_gwbridge
---


在docker v1.12版本中引入了swarm mode，同时出现了docker_gwbridge这个新的网桥，这个网桥的作用其实就是提供docker容器的对外访问的，因为在swarm mode下，假如容器使用了overlay网络，容器是不可以访问网络的，需要再给容器添加一张网卡，这张网卡就桥接在docker_gwbridge上，你去容器里面会看到，容器有两张网卡，这个过程是自动的。

对于docker0,我们可以利用--bip制定docker0的子网地址，官方没提供如何自定义dokcer_gwbridge的网段的方法，那么如何自定义dokcer_gwbridge网段了?

假如你执行了docker swarm init. docker_gwbridge会取一个默认的网段(172.18.0.0)，这时又该如何删除了？

```
docker swarm leave --force (让节点离开swarm mode)

docker network disconnect docker_gwbridge gateway_ingress-sbox -f (需要加-f，否则会说找不到这个容器)

docker network  rm docker_gwbridge

docker network create --subnet 192.168.2.0/24 --opt com.docker.network.bridge.name=docker_gwbridge --opt com.docker.network.bridge.enable_icc=false --opt com.docker.network.bridge.enable_ip_masquerade=true docker_gwbridge

docker swarm init
```


