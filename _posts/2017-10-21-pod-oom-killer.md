---
layout: post
author: shalou
title: "kubernetes的oom killer"
category: 容器技术
tag: [kubernetes, pod, qos, oom-killer]
---

kubenretes中的oom killer机制，指当宿主机资源不足时，内核为了维护自身的稳定性，会杀死某些进程（其实就是Containaer）

准确来说，oom killer其实是内核自我保护的机制，而kubernetes通过设置oomScore参数，可以调节oom的优先次序

## 1. docker中的oom参数

kubernetes中的oom killer其实是依赖docker提供的

<!-- more -->

``` golang
docker help run | grep cpu

  --cpu-percent int             CPU percent (Windows only)
  --cpu-period int              Limit CPU CFS (Completely Fair Scheduler) period
  --cpu-quota int               Limit CPU CFS (Completely Fair Scheduler) quota
  -c, --cpu-shares int              CPU shares (relative weight)
  --cpuset-cpus string          CPUs in which to allow execution (0-3, 0,1)
  --cpuset-mems string          MEMs in which to allow execution (0-3, 0,1)
docker help run | grep oom

  --oom-kill-disable            Disable OOM Killer
  --oom-score-adj int           Tune host's OOM preferences (-1000 to 1000)
docker help run | grep memory

  --kernel-memory string        Kernel memory limit
  -m, --memory string               Memory limit
  --memory-reservation string   Memory soft limit
  --memory-swap string          Swap limit equal to memory plus swap: '-1' to enable unlimited swap
  --memory-swappiness int       Tune container memory swappiness (0 to 100) (default -1)
```

其中有两个参数，即--oom-kill-disable和--oom-score-adj，--oom-kill-disable用于禁止oom killer，而--oom-score-adj用于调节score，score的值位于(-1000，1000)区间，值越大，则越有可能被kill

## 2. Pod Qos如何影响oom-killer

上一篇博客提到过，Pod Qos会影响oom-killer，Qos等级越低，则Container越容器被杀掉，我们通过源码看看kubernetes是如何影响oom-killer的

```golang
const (
    PodInfraOOMAdj        int = -998
    KubeletOOMScoreAdj    int = -999
    DockerOOMScoreAdj     int = -999
    KubeProxyOOMScoreAdj  int = -999
    guaranteedOOMScoreAdj int = -998
    besteffortOOMScoreAdj int = 1000
)

func GetContainerOOMScoreAdjust(pod *v1.Pod, container *v1.Container, memoryCapacity int64) int {
    switch v1qos.GetPodQOS(pod) {
    case v1.PodQOSGuaranteed:
        // Guaranteed containers should be the last to get killed.
        return guaranteedOOMScoreAdj
    case v1.PodQOSBestEffort:
        return besteffortOOMScoreAdj
    }   

    memoryRequest := container.Resources.Requests.Memory().Value()
    oomScoreAdjust := 1000 - (1000*memoryRequest)/memoryCapacity
    
    if int(oomScoreAdjust) < (1000 + guaranteedOOMScoreAdj) {
        return (1000 + guaranteedOOMScoreAdj)
    }
       
    if int(oomScoreAdjust) == besteffortOOMScoreAdj {
        return int(oomScoreAdjust - 1)
    }   
    return int(oomScoreAdjust)
}
```

我们前面提到过docker的oom-score-adj参数，该值越高，则容器越有可能被杀掉，代码中设置了几个常量。即PodInfraOOMAdj、KubeletOOMScoreAdj、DockerOOMScoreAdj、KubeProxyOOMScoreAdj、guaranteedOOMScoreAdj、 besteffortOOMScoreAdj

* PodInfraOOMAdj：假如容器是一个Pod Infra container，即为Pod提供network namespace的，该的oomAdj值为-998，因为该容器占用的资源本身就很少，即使kill，也回收不了多少资源，而且该容器对于Pod来说是至关重要的，因此它的值设置为-998
* KubeletOOMScoreAdj：kubelet当然是最重要的，因此它的值设置为-999
* DockerOOMScoreAdj：docker底层的基石，它的值位置为-999
* KubeProxyOOMScoreAdj：kube-proxy，与kubelet类似
* guaranteedOOMScoreAdj：即假如Pod的Qos为Guaranteed，则oomScore值为-998
* besteffortOOMScoreAdj：即假如Pod的Qos为BestEfford，则oomScore为1000，最容易被杀掉


从前面的分析我们已经获知当Pod的Qos为Guaranteed和BestEfford时oomScore的值，那当Pod的Qos为Burstable时，OomScore为多少呢？

我们可以看看GetContainerOOMScoreAdjust函数实现

memoryCapacity参数为node节点的总内存

```golang
oomScoreAdjust := 1000 - (1000*memoryRequest)/memoryCapacity

//oomScoreAdjust最小只能为1
if int(oomScoreAdjust) < (1000 + guaranteedOOMScoreAdj) {
        return (1000 + guaranteedOOMScoreAdj)
}

//oomScoreAdjust最小只能为999，必须低于besteffortOOMScoreAdj
if int(oomScoreAdjust) == besteffortOOMScoreAdj {
        return int(oomScoreAdjust - 1)
} 
```

即当容器的MemoryRequest越高，则oomScoreAdjust的值越小，被kill的机率越小


## 3. 总结
从上面的分析，我们可以看出Pod的Qos等级会影响Pod被Kill的概率。当内存紧张时，首先kill掉BestErrord；然后Kill掉Burstable，其中memoryRequest越大，则被kill的概率越少；最后才会选择kill吊Guaranteed
