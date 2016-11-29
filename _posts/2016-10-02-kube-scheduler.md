---
layout: post
author: shalou
title:  "kube-scheduler剖析"
category: 容器技术
tag: [k8s,kubernetes,kube-scheduler]
---

# 

kube-scheduler是k8s中的调度模块，负责调度Pod，pod是k8s最基本的运行单元。再我们剖析代码之前，可以猜测一下k8s是如何工作的。kube-scheduler需要对未被调度的Pod进行Watch，同时也需要对node进行watch，因为pod需要绑定到具体的Node上，当kube-scheduler监测到未被调度的pod，它会取出这个pod，然后依照内部设定的调度算法，选择合适的node，然后通过apiserver写回到etcd，kube-scheduler的工作也就结束了。实际kube-scheduler的代码就是这样工作的。

## 一、重要结构体

**AlgorithmProvider**：字面意思，调度算法提供者，包含两个sets.String，即FitPredicateKeys和PriorityFunctionKeys，FitPredicateKeys背后代表了一组过滤函数，PriorityFunctionKeys背后代表了一组打分加权函数。kube-scheduler初始化时，会默认初始化一个DefaultProvider。

```go
type AlgorithmProviderConfig struct {
    FitPredicateKeys     sets.String
    PriorityFunctionKeys sets.String
}
```
**Algorithm**：一个接口，需要实现Schedule函数即可，负责具体的调度算法

<!-- more -->

```go
type ScheduleAlgorithm interface {
    Schedule(*api.Pod, NodeLister) (selectedMachine string, err error)
}
```

**genericScheduler**：genericScheduler是一个具体的Algorithm，它实现了Schedule()函数。

```go
type genericScheduler struct {
    cache             schedulercache.Cache
    predicates        map[string]algorithm.FitPredicate
    prioritizers      []algorithm.PriorityConfig
    extenders         []algorithm.SchedulerExtender
    pods              algorithm.PodLister
    lastNodeIndexLock sync.Mutex
    lastNodeIndex     uint64
}
```

**FitPredicate**：过滤函数，返回值为bool，即一个节点要不符合要求，要不不符合要求

```go
type FitPredicate func(pod *api.Pod, nodeInfo *schedulercache.NodeInfo) (bool, error)
```
**PriorityFunction**：打分加权函数，前面可能会过滤多个node，这个函数给这些节点打分，返回各个节点的加权值。

```go
type PriorityFunction func(pod *api.Pod, nodeNameToInfo map[string]*schedulercache.NodeInfo, nodeLister NodeLister) (schedulerapi.HostPriorityList, error)
```
## 二、主要流程

函数入口，kube-scheduler本质上作为一个插件为k8s服务,因此他的代码入口位于plugin/cmd/kube-scheduler目录下。代码的结构和其他组件一致，首先创建一个server，然后解析命令行参数，最后运行起来。这里就不展开介绍。我们直接看Run函数

**plugin/cmd/kube-scheduler/scheduler.go +36**

```go
func main() {   
    runtime.GOMAXPROCS(runtime.NumCPU())
    s := options.NewSchedulerServer()
    s.AddFlags(pflag.CommandLine)

    flag.InitFlags()
    util.InitLogs()
    defer util.FlushLogs()

    verflag.PrintAndExitIfRequested()

    app.Run(s)
}
```

函数首先会构建一个configFactory，一个工厂类。我们进去看看NewConfigFactory函数，他构建了一些ListerAndWatcher，主要是为了从apiserver这里获取资源信息，它不仅获取了pod，node，还获取了PV、PVC、Service、controller(其实是replication controller)、replicaset这些信息后面都用得到。还有两个重要成员PodQueue和PodLister,PodQueue里面有未调度的Pod,而PodLister=schedulerCache，它负责从apiserver同步未调度的Pod，而且还保存了Pod调度过程中的状态变化。


**plugin/cmd/kube-scheduler/app/server.go +71**

```go
func Run(s *options.SchedulerServer) error {
    ...
    configFactory := factory.NewConfigFactory(kubeClient, s.SchedulerName, s.HardPodAffinitySymmetricWeight, s.FailureDomains)
    config, err := createConfig(s, configFactory)
    ...
    sched := scheduler.New(config)

    run := func(_ <-chan struct{}) {
        sched.Run()
        select {}
    }

    if !s.LeaderElection.LeaderElect {
        run(nil)
        glog.Fatal("this statement is unreachable")
        panic("unreachable")
    }
    ...
    //高可用，当有多个kube-scheduler时，确保只有一个能运行。
    leaderelection.RunOrDie(leaderelection.LeaderElectionConfig{
        EndpointsMeta: api.ObjectMeta{
            Namespace: "kube-system",
            Name:      "kube-scheduler",
        },
        Client:        kubeClient,
        Identity:      id,
        EventRecorder: config.Recorder,
        LeaseDuration: s.LeaderElection.LeaseDuration.Duration,
        RenewDeadline: s.LeaderElection.RenewDeadline.Duration,
        RetryPeriod:   s.LeaderElection.RetryPeriod.Duration,
        Callbacks: leaderelection.LeaderCallbacks{
            OnStartedLeading: run,
            OnStoppedLeading: func() {
                glog.Fatalf("lost master")
            },
        },
    })
    ...
}
```
**plugin/pkg/scheduler/factory/factory.go  +98**                                                                                                           

```go
func NewConfigFactory(client *client.Client, schedulerName string, hardPodAffinitySymmetricWeight int, failureDomains string) *ConfigFactory {
    stopEverything := make(chan struct{})
    schedulerCache := schedulercache.New(30*time.Second, stopEverything)

    c := &ConfigFactory{
        Client:             client,
        PodQueue:           cache.NewFIFO(cache.MetaNamespaceKeyFunc),
        ScheduledPodLister: &cache.StoreToPodLister{},
        // Only nodes in the "Ready" condition with status == "True" are schedulable
        NodeLister:                     &cache.StoreToNodeLister{},
        PVLister:                       &cache.StoreToPVFetcher{Store: cache.NewStore(cache.MetaNamespaceKeyFunc)},
        PVCLister:                      &cache.StoreToPVCFetcher{Store: cache.NewStore(cache.MetaNamespaceKeyFunc)},
        ServiceLister:                  &cache.StoreToServiceLister{Store: cache.NewStore(cache.MetaNamespaceKeyFunc)},
        ControllerLister:               &cache.StoreToReplicationControllerLister{Indexer: cache.NewIndexer(cache.MetaNamespaceKeyFunc, cache.Indexers{cache.NamespaceIndex: cache.MetaNamespaceIndexFunc})},
        ReplicaSetLister:               &cache.StoreToReplicaSetLister{Store: cache.NewStore(cache.MetaNamespaceKeyFunc)},
        schedulerCache:                 schedulerCache,
        StopEverything:                 stopEverything,
        SchedulerName:                  schedulerName,
        HardPodAffinitySymmetricWeight: hardPodAffinitySymmetricWeight,
        FailureDomains:                 failureDomains,
    }   

    c.PodLister = schedulerCache

    ...

    return c
}
```

继续看Run，createConfig(s, configFactory)，configFactory包含创建一个调度器必须的数据结构,主要是一些ListerAndWatch，用来watch资源，调度算法依赖于这些资源。接着我们用它来创建一个scheduler config，我们看看createConfig函数。其实函数主要分为两条路径，一条是configFactory.CreateFromConfig(policy)，另一条是configFactory.CreateFromProvider(s.AlgorithmProvider)。当存在PolicyConfigFile时（需要用户启动前传递）,前者，假如没有的话，则用s.AlgorithmProvider。这里的AlgorithmProvider默认为DefaultProvider。

那么什么是PolicyConfigFile,其实它主要包含一些过滤函数和打分加权函数，高级的用户可以自己组合这些函数，如下面代码所示。DefaultProvider其实本质上也是包含一些默认的过滤函数和打分加权函数。系统初始化就确定了

**plugin/cmd/kube-scheduler/app/server.go +163**

```go
func createConfig(s *options.SchedulerServer, configFactory *factory.ConfigFactory) (*scheduler.Config, error) {
    if _, err := os.Stat(s.PolicyConfigFile); err == nil {
        var (
            policy     schedulerapi.Policy
            configData []byte
        )   
        configData, err := ioutil.ReadFile(s.PolicyConfigFile)
        if err != nil {
            return nil, fmt.Errorf("unable to read policy config: %v", err)
        }   
        if err := runtime.DecodeInto(latestschedulerapi.Codec, configData, &policy); err != nil {
            return nil, fmt.Errorf("invalid configuration: %v", err)
        }   
        return configFactory.CreateFromConfig(policy)
    }   

    // if the config file isn't provided, use the specified (or default) provider
    return configFactory.CreateFromProvider(s.AlgorithmProvider)
}
```
用户自定义的Policy函数，它包含了PodFitsPorts、PodFitsResources、NoDiskConflict等过滤函数，LeastRequestedPriority、BalancedResourceAllocation等加权函数。

```json

{
"kind" : "Policy",
"apiVersion" : "v1",
"predicates" : [
	{"name" : "PodFitsPorts"},
	{"name" : "PodFitsResources"},
	{"name" : "NoDiskConflict"},
	{"name" : "NoVolumeZoneConflict"},
	{"name" : "MatchNodeSelector"},
	{"name" : "HostName"}
	],
"priorities" : [
	{"name" : "LeastRequestedPriority", "weight" : 1},
	{"name" : "BalancedResourceAllocation", "weight" : 1},
	{"name" : "ServiceSpreadingPriority", "weight" : 1},
	{"name" : "EqualPriority", "weight" : 1}
	]
}
```

**plugin/pkg/scheduler/algorithmprovider/defaults/defaults.go +55**

init()函数在main运行前就会运行，它会提前将DefaultProvider、FitPredicate(过滤)、Priority(打分加权)注册到factory里面,而DefaultProvider有默认的defaultPredicates()、defaultPriorities()，这两个函数我们没有粘贴出来。

```go
func init() {
    factory.RegisterAlgorithmProvider(factory.DefaultProvider, defaultPredicates(), defaultPriorities())
    factory.RegisterPriorityConfigFactory(
        "ServiceSpreadingPriority",
        factory.PriorityConfigFactory{
            Function: func(args factory.PluginFactoryArgs) algorithm.PriorityFunction {
                return priorities.NewSelectorSpreadPriority(args.PodLister, args.ServiceLister, algorithm.EmptyControllerLister{}, algorithm.EmptyReplicaSetLister{})
            },
            Weight: 1,
        },
    )
    factory.RegisterFitPredicate("PodFitsPorts", predicates.PodFitsHostPorts)
    factory.RegisterPriorityFunction("ImageLocalityPriority", priorities.ImageLocalityPriority, 1)
    factory.RegisterFitPredicate("PodFitsHostPorts", predicates.PodFitsHostPorts)
    factory.RegisterFitPredicate("PodFitsResources", predicates.PodFitsResources)
    factory.RegisterFitPredicate("HostName", predicates.PodFitsHost)
    factory.RegisterFitPredicate("MatchNodeSelector", predicates.PodSelectorMatches)
    factory.RegisterFitPredicateFactory(
        "MatchInterPodAffinity",
        func(args factory.PluginFactoryArgs) algorithm.FitPredicate {
            return predicates.NewPodAffinityPredicate(args.NodeInfo, args.PodLister, args.FailureDomains)
        },
    )
    factory.RegisterPriorityConfigFactory(
        "InterPodAffinityPriority",
        factory.PriorityConfigFactory{
            Function: func(args factory.PluginFactoryArgs) algorithm.PriorityFunction {
                return priorities.NewInterPodAffinityPriority(args.NodeInfo, args.NodeLister, args.PodLister, args.HardPodAffinitySymmetricWeight, args.FailureDomains)
            },
            Weight: 1,
        },
    )
}
```

无论是从policyfile，还是从algorithmprovider创建scheduler，最后都会调用一个函数NewGenericScheduler。这个函数构建了一个真正的调度器。

```go
func NewGenericScheduler(cache schedulercache.Cache, predicates map[string]algorithm.FitPredicate, prioritizers []algorithm.PriorityConfig, extenders []algorithm.SchedulerExtender) algorithm.ScheduleAlgorithm {
    return &genericScheduler{
        cache:       cache, 
        predicates:   predicates,
        prioritizers: prioritizers,
        extenders:    extenders,
    }       
}
```
他用cache、predicates 、prioritizers、extender构建了一个genericScheduler,genericScheduler类型为algorithm.ScheduleAlgorithm，准确说应该是一个调度器算法。


回到开头的Run函数,sched.Run()运行调度器。

**plugin/pkg/scheduler/scheduler.go +89**

```go
func (s *Scheduler) Run() {
    go wait.Until(s.scheduleOne, 0, s.config.StopEverything)
}

func (s *Scheduler) scheduleOne() {
    pod := s.config.NextPod() 
    ...
    dest, err := s.config.Algorithm.Schedule(pod, s.config.NodeLister)
    ...
    assumed := *pod
    assumed.Spec.NodeName = dest
    if err := s.config.SchedulerCache.AssumePod(&assumed); err != nil {
        glog.Errorf("scheduler cache AssumePod failed: %v", err)
    }

    go func() {
        ...
        b := &api.Binding{
            ObjectMeta: api.ObjectMeta{Namespace: pod.Namespace, Name: pod.Name},
            Target: api.ObjectReference{
                Kind: "Node",
                Name: dest,
            },
        }

        bindingStart := time.Now()
        err := s.config.Binder.Bind(b)
        ...
    }()
}
```

看到这个函数应该很清晰了，首先pod := s.config.NextPod()获取一个未调度的pod，然后s.config.Algorithm.Schedule(pod, s.config.NodeLister)，即用算法为pod选择一个合适的node，最后s.config.Binder.Bind(b)绑定pod到node上去。

我们看看调度算法是如何选择具体的node的。首先findNodesThatFit过滤节点，PrioritizeNodes给过滤的节点打分，g.selectHost挑选出分数最高的节点并返回。这些函数我们就不详细展开了。这样就完成了pod的调度。

**plugin/pkg/scheduler/generic_scheduler.go +70**

```go
func (g *genericScheduler) Schedule(pod *api.Pod, nodeLister algorithm.NodeLister) (string, error) {
    ...
    nodes, err := nodeLister.List()
    ...
    nodeNameToInfo, err := g.cache.GetNodeNameToInfoMap()
    ...
    filteredNodes, failedPredicateMap, err := findNodesThatFit(pod, nodeNameToInfo, g.predicates, nodes, g.extenders)
    ...
    priorityList, err := PrioritizeNodes(pod, nodeNameToInfo, g.prioritizers, algorithm.FakeNodeLister(filteredNodes), g.extenders)
    ...
    return g.selectHost(priorityList)
}
```

## 三、Factory作用
Factory包含了众多的ListerAndWatcher，主要是为了watch apiserver，获取最新的资源。创建Factory时，并没有立即启动Factory，而是在创建scheduler config时，启动了这个Factory。

**plugin/pkg/scheduler/factory/factory.go +385**

```go
func (f *ConfigFactory) Run() {
    cache.NewReflector(f.createUnassignedNonTerminatedPodLW(), &api.Pod{}, f.PodQueue, 0).RunUntil(f.StopEverything)

    go f.scheduledPodPopulator.Run(f.StopEverything)

    go f.nodePopulator.Run(f.StopEverything)

    cache.NewReflector(f.createPersistentVolumeLW(), &api.PersistentVolume{}, f.PVLister.Store, 0).RunUntil(f.StopEverything)
    cache.NewReflector(f.createPersistentVolumeClaimLW(), &api.PersistentVolumeClaim{}, f.PVCLister.Store, 0).RunUntil(f.StopEverything)

    cache.NewReflector(f.createServiceLW(), &api.Service{}, f.ServiceLister.Store, 0).RunUntil(f.StopEverything)
    
    cache.NewReflector(f.createControllerLW(), &api.ReplicationController{}, f.ControllerLister.Indexer, 0).RunUntil(f.StopEverything)

    cache.NewReflector(f.createReplicaSetLW(), &extensions.ReplicaSet{}, f.ReplicaSetLister.Store, 0).RunUntil(f.StopEverything)
}
```
这里创建了6个reflector，还有两个在创建Factory时就创建好的reflector：scheduledPodPopulator和nodePopulator。然后启动了这些reflector。我们来分析这些reflector。



