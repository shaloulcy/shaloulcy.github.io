---
layout: post 
author: shalou
title:  "keystone配置redis存储token" 
category: openstack
tag: [openstack, redis, keystone]
---


```golang
[token]
provider = uuid
driver = kvs
caching = true

[cache]
enabled=true
backend=dogpile.cache.redis
backend_argument=url:redis://127.0.0.1:6379/2
expiration_time = 600
```
