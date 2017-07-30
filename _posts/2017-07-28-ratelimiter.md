---
layout: post
author: shalou
title: "流控算法" 
category: web
tag: [web, ratelimiter]
---

# 流控算法

流控算法分为两种，一种是令牌桶(token bucket)算法，另一种是漏桶算法(leak bucket)

## 1. 令牌桶算法

令牌桶一般有两个参数：QPS和BURST。QPS为每秒查询数，而BURST指每秒最大并发数。

<!-- more -->

```go
//其中capacity表示BURST，而fillInterval表示1e9/QPS
type TokenBucket{
	startTime    time.Time
	capacity     int64     
	fillInterval time.Duration //加令牌的间隔
	mu sync.Mutex
	avail     int64 //桶中剩余令牌
}

func (tb *TokenBucket) Take() bool{
	tb.mu.Lock()
	defer tb.mu.Unlock()

    now:=time.Now()
    add := int64(now.Sub(tb.startTime)/tb.fillInterval)
    if add > 0 {
        tb.avail += add //增加令牌
        if tb.avail > tb.capacity {
            tb.avail = tb.capacity
        }
        tb.startTime=now
    }

    if tb.avail > 0 {
        tb.avail -= 1 //获取令牌
        retun true
    }
    return false  //拒绝
}
```

# 2. 漏桶算法

漏桶算法相比令牌桶算法只有QPS，没有BURST，流量不断进入桶内，假如桶满了流量则会溢出，桶中流量以恒定的速率往外流。可以用队列、定时器的形式实现。

漏桶算法强制一个常量的输出速率而不管输入数据流的突发性。当输入空闲时，该算法不执行任何动作

令牌队列适合有突发流量的情况
