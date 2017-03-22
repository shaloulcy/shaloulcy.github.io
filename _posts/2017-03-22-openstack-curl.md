---
layout: post 
author: shalou
title:  "curl访问OpenStack" 
category: openstack
tag: [openstack, curl]
---

任何一次访问都需要经过先经过认证，因此我们首先需要向Keystone组件进行认证，获取Token

<!-- more -->

```python
curl -i -g -X POST -H "Content-Type: application/json" http://openstack:35357/v3/auth/tokens '{
    "auth": {
        "identity": {
            "methods": [
                "password"
            ],  
            "password": {
                "user": {
                    "name": "admin",
                    "password": "admin",
                    "domain": {"id": "default"}
                }   
            }   
        },  
        "scope": {
            "project": {
                 "name": "admin",
                 "domain": {"id": "default"}
            }   
        }   
    }   
}'


返回值如下

HTTP/1.1 201 Created
X-Subject-Token: dabdd91adf56440eb5a1e9a305c74bc3
Vary: X-Auth-Token
Content-Type: application/json
Content-Length: 2790
X-Openstack-Request-Id: req-275258bb-bb23-41b1-b749-5e82c34211bb
Date: Tue, 21 Mar 2017 23:29:32 GMT
{"token": {"methods": ["password"], "roles": [{"id": "cd0bdd4a14bb4c5f830b97f83520de8f", "name": "admin"}], "expires_at": "2017-03-22T00:29:32.953582Z", "project": {"domain": {"id": "default", "name": "Default"}, "id": "c08977f8313543688b666ce844b7c71a", "name": "admin"}, "catalog": [{"endpoints": [{"region_id": "RegionOne", "url": "http://openstack:35357/v2.0", "region": "RegionOne", "interface": "admin", "id": "672844622d714e00b5cf97dec96f323c"}, {"region_id": "RegionOne", "url": "http://openstack:5000/v2.0", "region": "RegionOne", "interface": "internal", "id": "9e9c14f2448e4d04887eda5ee5b74d8b"}, {"region_id": "RegionOne", "url": "http://openstack:5000/v2.0", "region": "RegionOne", "interface": "public", "id": "a50fc8ef645f412a875fbdfd5dc3f623"}], "type": "identity", "id": "3102a68ddfa54862ae553e5eeae4a0f0", "name": "keystone"}, {"endpoints": [{"region_id": "RegionOne", "url": "http://openstack:9292", "region": "RegionOne", "interface": "internal", "id": "9a423aa84a6b4a33bbebeb429cf66408"}, {"region_id": "RegionOne", "url": "http://openstack:9292", "region": "RegionOne", "interface": "admin", "id": "c2c9c8afaa9644229f14a91f73212688"}, {"region_id": "RegionOne", "url": "http://openstack:9292", "region": "RegionOne", "interface": "public", "id": "cf311754971448aba6058d0935094b4e"}], "type": "image", "id": "a46cbf41f06041f6b00e902e7b5f3b7f", "name": "glance"}, {"endpoints": [{"region_id": "RegionOne", "url": "http://openstack:9696", "region": "RegionOne", "interface": "admin", "id": "096af89ec5fb4d95bc1bb268a5493e1b"}, {"region_id": "RegionOne", "url": "http://openstack:9696", "region": "RegionOne", "interface": "public", "id": "6ae0140bb4b241e4a127687cdc406854"}, {"region_id": "RegionOne", "url": "http://openstack:9696", "region": "RegionOne", "interface": "internal", "id": "d631e32ece1849e6a40607783e02dede"}], "type": "network", "id": "aba3389fd4c24437b1c7fc7895fc29e1", "name": "neutron"}, {"endpoints": [{"region_id": "RegionOne", "url": "http://openstack:8774/v2/c08977f8313543688b666ce844b7c71a", "region": "RegionOne", "interface": "internal", "id": "10bebe2d2b7543bd847bc1973629e49d"}, {"region_id": "RegionOne", "url": "http://openstack:8774/v2/c08977f8313543688b666ce844b7c71a", "region": "RegionOne", "interface": "admin", "id": "5b19ddbc91c942e4a7a76c22c477ff54"}, {"region_id": "RegionOne", "url": "http://openstack:8774/v2/c08977f8313543688b666ce844b7c71a", "region": "RegionOne", "interface": "public", "id": "ab05e096cd744eb6bba616aa8a430f57"}], "type": "compute", "id": "abdfb1d8b871446186dd7466837e5170", "name": "nova"}], "extras": {}, "user": {"domain": {"id": "default", "name": "Default"}, "id": "6fd9896c967e4c5cbb01546a0f1b31e0", "name": "admin"}, "audit_ids": ["2T2umIgkRA6VKzyKfdNZfg"], "issued_at": "2017-03-21T23:29:32.953625Z"}
```

认证后会在http返回头里面有X-Subject-Token: dabdd91adf56440eb5a1e9a305c74bc3，这个uuid就是返回的Token，同时还会返回一大堆各个服务的endpoints，如nova服务的endpoint为http://openstack:8774/v2/c08977f8313543688b666ce844b7c71a


接下来我们查看该用户有哪些虚拟机

```python
curl -i -g -X GET -H "Content-Type: application/json" -H "X-Auth-Token: dabdd91adf56440eb5a1e9a305c74bc3" http://openstack:8774/v2/c08977f8313543688b666ce844b7c71a/servers

返回值如下，有一台test虚拟机
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 351
X-Compute-Request-Id: req-b912d0fc-74e8-452f-b110-e60e1069f422
Date: Tue, 21 Mar 2017 23:39:25 GMT

{"servers": [{"id": "79f7154f-74ff-46bc-9b0c-1c7ba5563544", "links": [{"href": "http://openstack:8774/v2/c08977f8313543688b666ce844b7c71a/servers/79f7154f-74ff-46bc-9b0c-1c7ba5563544", "rel": "self"}, {"href": "http://openstack:8774/c08977f8313543688b666ce844b7c71a/servers/79f7154f-74ff-46bc-9b0c-1c7ba5563544", "rel": "bookmark"}], "name": "test"}]}
```

创建一台虚拟机

```python
curl -i -g -X POST -H "Content-Type: application/json" -H "X-Auth-Token: dabdd91adf56440eb5a1e9a305c74bc3" http://openstack:8774/v2/c08977f8313543688b666ce844b7c71a/servers -d '{
    "server": {
        "name": "demo",
        "imageRef": "28940ff2-59f7-441d-9314-5eb6152e851d",
        "availability_zone": "nova",
        "flavorRef": "1",
        "max_count": 1,
        "min_count": 1,
        "networks": [
            {
                "uuid": "6df5c1c3-ddd8-4206-b220-14de4e1757d0"
            }
        ],
        "security_groups": [
            {
                "name": "default"
            }
        ]
    }
}'

返回值
HTTP/1.1 202 Accepted
Location: http://openstack:8774/v2/c08977f8313543688b666ce844b7c71a/servers/a9dff147-8161-4d8c-8db8-9092466c6370
Content-Type: application/json
Content-Length: 434
X-Compute-Request-Id: req-4ff559d9-da30-4419-9b0d-007899253cda
Date: Tue, 21 Mar 2017 23:51:29 GMT

{"server": {"security_groups": [{"name": "default"}], "OS-DCF:diskConfig": "MANUAL", "id": "a9dff147-8161-4d8c-8db8-9092466c6370", "links": [{"href": "http://openstack:8774/v2/c08977f8313543688b666ce844b7c71a/servers/a9dff147-8161-4d8c-8db8-9092466c6370", "rel": "self"}, {"href": "http://openstack:8774/c08977f8313543688b666ce844b7c71a/servers/a9dff147-8161-4d8c-8db8-9092466c6370", "rel": "bookmark"}], "adminPass": "YkDYjh7SwaBf"}}
```

一般情况下我们都使用dashboard活着命令行client去操控OpenStack的资源，OpenStack提供你了各种语言的client，方便调用其API，但这些client本质上还是发送http请求的，因此我们必须清楚底层是如何发送http请求的，这也是本文的目的所在

OpenStack官方提供了专门的API文档，供我们详细查询
