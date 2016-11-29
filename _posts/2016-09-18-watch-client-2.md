---
layout: post
author: shalou
title:  "kube-apiserver的watch机制(Client端)(第2部分)"
category: 容器技术
tag: [k8s, kubernetes]
---

*本源码分析基于k8s v1.3.6*

kubelet对kube-apiserver的watch机制还是比较简陋，我们来看看kube-controller-manager是如何对kube-apiserver进行watch的。

## replication controller的实现
replication controller是kube-controller-manager里面一个重要的控制器，主要对rc进行控制，确保，pods的数量恰好和rc的规定一致。因此replication controller主要对这两类进行watch，一类是replication controller，另一类是pods。这两类资源刚好属于两个不同的范畴，pods是许多控制器共享的，像endpoint controller也需要对pods进行watch，而replication controller是独享的。因此对他们的watch机制来不一样，我们首先看看如何watch replication controller。

**cmd/kube-controller-manager/app/controllermanager.go +198**

<!-- more -->

```go
func StartControllers(s *options.CMServer, kubeClient *client.Client, kubeconfig *restclient.Config, stop <-chan struct{}) error {
    podInformer := informers.CreateSharedPodIndexInformer(clientset.NewForConfigOrDie(restclient.AddUserAgent(kubeconfig, "pod-informer")), ResyncPeriod(s)())
    nodeInformer := informers.CreateSharedNodeIndexInformer(clientset.NewForConfigOrDie(restclient.AddUserAgent(kubeconfig, "node-informer")), ResyncPeriod(s)())
    pvcInformer := informers.CreateSharedPVCIndexInformer(clientset.NewForConfigOrDie(restclient.AddUserAgent(kubeconfig, "pvc-informer")), ResyncPeriod(s)())
    pvInformer := informers.CreateSharedPVIndexInformer(clientset.NewForConfigOrDie(restclient.AddUserAgent(kubeconfig, "pv-informer")), ResyncPeriod(s)())
    informers := map[reflect.Type]framework.SharedIndexInformer{}
    informers[reflect.TypeOf(&api.Pod{})] = podInformer
    informers[reflect.TypeOf(&api.Node{})] = nodeInformer
    informers[reflect.TypeOf(&api.PersistentVolumeClaim{})] = pvcInformer
    informers[reflect.TypeOf(&api.PersistentVolume{})] = pvInformer

    go endpointcontroller.NewEndpointController(podInformer, clientset.NewForConfigOrDie(restclient.AddUserAgent(kubeconfig, "endpoint-controller"))).
        Run(int(s.ConcurrentEndpointSyncs), wait.NeverStop)
    time.Sleep(wait.Jitter(s.ControllerStartInterval.Duration, ControllerStartJitter))

    go replicationcontroller.NewReplicationManager(
        podInformer,
        clientset.NewForConfigOrDie(restclient.AddUserAgent(kubeconfig, "replication-controller")),
        ResyncPeriod(s),
        replicationcontroller.BurstReplicas,
        int(s.LookupCacheSizeForRC),
    ).Run(int(s.ConcurrentRCSyncs), wait.NeverStop)
    ...
}
```
首先创建了一个podInformer，创建的函数是informers.CreateSharedPodIndexInformer，从字面意思就知道podInformer是共享的，我们暂时不关注。查看函数replicationcontroller.NewReplicationManager

**pkg/controller/replication/replication_controller.go +111**

```go
func NewReplicationManager(podInformer framework.SharedIndexInformer, kubeClient clientset.Interface, resyncPeriod controller.ResyncPeriodFunc, burstReplicas int, lookupCacheSize int) *ReplicationManager {
    eventBroadcaster := record.NewBroadcaster()
    eventBroadcaster.StartLogging(glog.Infof)
    eventBroadcaster.StartRecordingToSink(&unversionedcore.EventSinkImpl{Interface: kubeClient.Core().Events("")})
    return newReplicationManagerInternal(
        eventBroadcaster.NewRecorder(api.EventSource{Component: "replication-controller"}),
        podInformer, kubeClient, resyncPeriod, burstReplicas, lookupCacheSize)
}

// newReplicationManagerInternal configures a replication manager with the specified event recorder
func newReplicationManagerInternal(eventRecorder record.EventRecorder, podInformer framework.SharedIndexInformer, kubeClient clientset.Interface, resyncPeriod controller.ResyncPeriodFunc, burstReplicas int, lookupCacheSize int) *ReplicationManager {
    if kubeClient != nil && kubeClient.Core().GetRESTClient().GetRateLimiter() != nil {
        metrics.RegisterMetricAndTrackRateLimiterUsage("replication_controller", kubeClient.Core().GetRESTClient().GetRateLimiter())
    }   

    rm := &ReplicationManager{
        kubeClient: kubeClient,
        podControl: controller.RealPodControl{
            KubeClient: kubeClient,
            Recorder:   eventRecorder,
        },  
        burstReplicas: burstReplicas,
        expectations:  controller.NewUIDTrackingControllerExpectations(controller.NewControllerExpectations()),
        queue:         workqueue.New(),
    }

    rm.rcStore.Indexer, rm.rcController = framework.NewIndexerInformer(
        &cache.ListWatch{
            ListFunc: func(options api.ListOptions) (runtime.Object, error) {
                return rm.kubeClient.Core().ReplicationControllers(api.NamespaceAll).List(options)
            },
            WatchFunc: func(options api.ListOptions) (watch.Interface, error) {
                return rm.kubeClient.Core().ReplicationControllers(api.NamespaceAll).Watch(options)
            },
        },
        &api.ReplicationController{},
        // TODO: Can we have much longer period here?
        FullControllerResyncPeriod,
        framework.ResourceEventHandlerFuncs{
            AddFunc: rm.enqueueController,
            UpdateFunc: func(old, cur interface{}) {
                oldRC := old.(*api.ReplicationController)
                curRC := cur.(*api.ReplicationController)

                if !reflect.DeepEqual(oldRC.Spec.Selector, curRC.Spec.Selector) {
                    rm.lookupCache.InvalidateAll()
                }

                if oldRC.Status.Replicas != curRC.Status.Replicas {
                    glog.V(4).Infof("Observed updated replica count for rc: %v, %d->%d", curRC.Name, oldRC.Status.Replicas, curRC.Status.Replicas)
                }
                rm.enqueueController(cur)
            },
            DeleteFunc: rm.enqueueController,
        },
        cache.Indexers{cache.NamespaceIndex: cache.MetaNamespaceIndexFunc},
    )

    podInformer.AddEventHandler(framework.ResourceEventHandlerFuncs{
        AddFunc: rm.addPod,
        UpdateFunc: rm.updatePod,
        DeleteFunc: rm.deletePod,
    })
    rm.podStore.Indexer = podInformer.GetIndexer()
    rm.podController = podInformer.GetController()

    rm.syncHandler = rm.syncReplicationController
    rm.podStoreSynced = rm.podController.HasSynced
    rm.lookupCache = controller.NewMatchingCache(lookupCacheSize)
    return rm
}
```

rm.rcStore.Indexer, rm.rcController = framework.NewIndexerInformer这个函数是点睛之笔，它是watch的实现。我们查看一下这个函数。

**pkg/controller/framework/controller.go +277**

```go
func NewIndexerInformer(
    lw cache.ListerWatcher,
    objType runtime.Object,
    resyncPeriod time.Duration,
    h ResourceEventHandler,
    indexers cache.Indexers,
) (cache.Indexer, *Controller) {
    clientState := cache.NewIndexer(DeletionHandlingMetaNamespaceKeyFunc, indexers)

    fifo := cache.NewDeltaFIFO(cache.MetaNamespaceKeyFunc, nil, clientState)

    cfg := &Config{
        Queue:            fifo,
        ListerWatcher:    lw, 
        ObjectType:       objType,
        FullResyncPeriod: resyncPeriod,
        RetryOnError:     false,

        Process: func(obj interface{}) error {
            // from oldest to newest
            for _, d := range obj.(cache.Deltas) {
                switch d.Type {
                case cache.Sync, cache.Added, cache.Updated:
                    if old, exists, err := clientState.Get(d.Object); err == nil && exists {
                        if err := clientState.Update(d.Object); err != nil {
                            return err
                        }
                        h.OnUpdate(old, d.Object)
                    } else {
                        if err := clientState.Add(d.Object); err != nil {
                            return err
                        }
                        h.OnAdd(d.Object)
                    }
                case cache.Deleted:
                    if err := clientState.Delete(d.Object); err != nil {
                        return err
                    }
                    h.OnDelete(d.Object)
                }
            }
            return nil
        },
    }
    return clientState, New(cfg)
}
```

这个函数有两个返回结果，cache.Indexer和*Controller，输入参数为五个分别为lw、objectType、resyncPeriod、h和indexers，类型分别为cache.ListerWatcher、runtime.Object、time.Duration、ResourceEventHandler和cache.Indexers。lw其实就是一个ListWatch，包含List函数和Watch函数，objectType表示要watch的资源类型，resyncPeriod是个时间间隔，h是一个触发函数，当watch到资源以后，用这个函数去处理，它包括OnAdd、onUpdate和onUpdate三个函数，根据watch的类型相应触发这些函数。

该函数该生成一个config，config包装了lw、objectType，还有一个cache,cache的类型为DeltaFIFO,即是一个队列。此外好包括一个Process函数。然后几个这个Config创建一个控制器

**pkg/controller/framework/controller.go +78**

```go
func New(c *Config) *Controller {
    ctlr := &Controller{
        config: *c, 
    }   
    return ctlr
}

func (c *Controller) Run(stopCh <-chan struct{}) {
    defer utilruntime.HandleCrash()
    r := cache.NewReflector(
        c.config.ListerWatcher,
        c.config.ObjectType,
        c.config.Queue,
        c.config.FullResyncPeriod,
    )   

    c.reflectorMutex.Lock()
    c.reflector = r 
    c.reflectorMutex.Unlock()

    r.RunUntil(stopCh)

    wait.Until(c.processLoop, time.Second, stopCh)
}
```

控制器的结构比较简单，只传入了Config，我们可以看看控制器Run以后干的事情。他首先构建了一个reflector!!!是不是很熟悉，在前面的文章中，我们反复强调了这个结构体，就是又这个结构体实现watch的。informer只是包装了reflector。reflector的几个传入参数：c.config.ListerWatcher、c.config.ObjectType和c.config.Queue,Queue就是一个Cache。从之前的分析我们可以看到运行这个reflector以后，会调用ListerWatcher,然后把结果传入Cache。我们来看看这个Cache有什么特殊

**pkg/client/cache/delta_fifo.go**

```go
type DeltaFIFO struct {
    items map[string]Deltas
    queue []string

    initialPopulationCount int 

    keyFunc KeyFunc
    
    deltaCompressor DeltaCompressor

    knownObjects KeyListerGetter
}

func (f *DeltaFIFO) Add(obj interface{}) error {
    f.lock.Lock()
    defer f.lock.Unlock()
    f.populated = true   
    return f.queueActionLocked(Added, obj)
}

func (f *DeltaFIFO) Update(obj interface{}) error {
    f.lock.Lock()
    defer f.lock.Unlock()
    f.populated = true   
    return f.queueActionLocked(Updated, obj)
}

func (f *DeltaFIFO) Delete(obj interface{}) error {
    id, err := f.KeyOf(obj)   
    if err != nil {      
        return KeyError{obj, err} 
    }  
    f.lock.Lock()        
    defer f.lock.Unlock()
    f.populated = true
    if f.knownObjects == nil {
        if _, exists := f.items[id]; !exists {
            return nil
        }
    } else {
        _, exists, err := f.knownObjects.GetByKey(id)
        _, itemsExist := f.items[id]
        if err == nil && !exists && !itemsExist {
            return nil
        }
    }

    return f.queueActionLocked(Deleted, obj)
}

func (f *DeltaFIFO) queueActionLocked(actionType DeltaType, obj interface{}) error {
    id, err := f.KeyOf(obj)
    if err != nil {
        return KeyError{obj, err}
    }   
    
    if actionType == Sync && f.willObjectBeDeletedLocked(id) {
        return nil 
    }   
        
    newDeltas := append(f.items[id], Delta{actionType, obj})
    newDeltas = dedupDeltas(newDeltas)
    if f.deltaCompressor != nil {
        newDeltas = f.deltaCompressor.Compress(newDeltas)
    }       
        
    _, exists := f.items[id]
    if len(newDeltas) > 0 {
        if !exists {
            f.queue = append(f.queue, id)
        }
        f.items[id] = newDeltas
        f.cond.Broadcast()
    } else if exists {
        delete(f.items, id)
    }
    return nil
}
```
处理的结果会更新到f.items里面。回到前面的函数，reflector启动以后，程序接着运行了wait.Until(c.processLoop, time.Second, stopCh)。我们看看processLoop函数

**pkg/controller/framework/controller.go +121**

```go
func (c *Controller) processLoop() {
    for {
        obj, err := c.config.Queue.Pop(cache.PopProcessFunc(c.config.Process))
        if err != nil {  
            if c.config.RetryOnError {
                // This is the safe way to re-enqueue.
                c.config.Queue.AddIfNotPresent(obj)
            }            
        }
    }                    
}
```
这是一个典型的生产者和消费者模型，reflector往fifo里面添加数据，而processLoop就不停去消费这里这些数据。cache.PopProcessFunc(c.config.Process)将前面Process函数传递进去。我们看看Pop操作

**pkg/client/cache/delta_fifo.go**

```go
func (f *DeltaFIFO) Pop(process PopProcessFunc) (interface{}, error) {
    f.lock.Lock()
    defer f.lock.Unlock()
    for {
        for len(f.queue) == 0 {   
            f.cond.Wait()
        }
        id := f.queue[0] 
        f.queue = f.queue[1:]     
        item, ok := f.items[id]   
        if f.initialPopulationCount > 0 {
            f.initialPopulationCount--
        }
        if !ok {
            // Item may have been deleted subsequently.
            continue     
        }
        delete(f.items, id)       
        // Don't need to copyDeltas here, because we're transferring
        // ownership to the caller.
        return item, process(item)
    }  
}
```
从函数可以看出，主要从f.items取出object，然后调用process函数进行处理。再创建informer时，就指定了Process函数，如下所示。

**pkg/controller/framework/controller.go +277**

```go
func NewIndexerInformer(
    lw cache.ListerWatcher,
    objType runtime.Object,
    resyncPeriod time.Duration,
    h ResourceEventHandler,
    indexers cache.Indexers,
) (cache.Indexer, *Controller) {
    clientState := cache.NewIndexer(DeletionHandlingMetaNamespaceKeyFunc, indexers)

    fifo := cache.NewDeltaFIFO(cache.MetaNamespaceKeyFunc, nil, clientState)

    cfg := &Config{
        Queue:            fifo,
        ListerWatcher:    lw, 
        ObjectType:       objType,
        FullResyncPeriod: resyncPeriod,
        RetryOnError:     false,

        Process: func(obj interface{}) error {
            // from oldest to newest
            for _, d := range obj.(cache.Deltas) {
                switch d.Type {
                case cache.Sync, cache.Added, cache.Updated:
                    if old, exists, err := clientState.Get(d.Object); err == nil && exists {
                        if err := clientState.Update(d.Object); err != nil {
                            return err
                        }
                        h.OnUpdate(old, d.Object)
                    } else {
                        if err := clientState.Add(d.Object); err != nil {
                            return err
                        }
                        h.OnAdd(d.Object)
                    }
                case cache.Deleted:
                    if err := clientState.Delete(d.Object); err != nil {
                        return err
                    }
                    h.OnDelete(d.Object)
                }
            }
            return nil
        },
    }
    return clientState, New(cfg)
}
```

从函数我们可以看出，实际上obj被存储在clientState里面。在对obje进行Add、Update、Delete时，会触发onAdd、onUpdate、onDelete。最终他们落到一下几个函数上面

**pkg/controller/replication/replication_controller.go +149**

```go
framework.ResourceEventHandlerFuncs{
    AddFunc: rm.enqueueController,
    UpdateFunc: func(old, cur interface{}) {
        oldRC := old.(*api.ReplicationController)
        curRC := cur.(*api.ReplicationController)

        if !reflect.DeepEqual(oldRC.Spec.Selector, curRC.Spec.Selector) {
              rm.lookupCache.InvalidateAll()
        }

        f oldRC.Status.Replicas != curRC.Status.Replicas {
        glog.V(4).Infof("Observed updated replica count for rc: %v, %d->%d", curRC.Name, oldRC.Status.Replicas, curRC.Status.Replicas)
        }
        rm.enqueueController(cur)
    },
    DeleteFunc: rm.enqueueController,
}
```
obj最终会调用rm.enqueueController,会把obj的key假如到replicationmanager的queue里面。注意好像删除的obj也会调用这个enqueueController函数

**pkg/controller/replication/replication_controller.go**

```go
func (rm *ReplicationManager) enqueueController(obj interface{}) {
    key, err := controller.KeyFunc(obj)
    if err != nil {
        glog.Errorf("Couldn't get key for object %+v: %v", obj, err)
        return
    }   
    rm.queue.Add(key)
}
```

又是一个生产者与消费者模型，replicationmanager创建了五个worker去消费添加的key。syncHandler是个重要的函数，由他负责pod与rc的同步，确保Pod副本数与rc规定的相同。

**pkg/controller/replication/replication_controller.go**

```go
func (rm *ReplicationManager) worker() {
    workFunc := func() bool { 
        key, quit := rm.queue.Get()
        if quit {
            return true
        }
        defer rm.queue.Done(key)  
        err := rm.syncHandler(key.(string))
        if err != nil {  
            glog.Errorf("Error syncing replication controller: %v", err)
        }
        return false
    }  
    for {
        if quit := workFunc(); quit {
            glog.Infof("replication controller worker shutting down")
            return       
        }
    }  
}
```

##对Pod的Watch
文章首页提到过，kube-controller-manager会创建一个podinformer，这个informer是共享的，又多个controller进行共享。多个controller如何共享这个informer。看如下代码

**pkg/controller/replication/replication_controller.go +203**

```go
podInformer.AddEventHandler(framework.ResourceEventHandlerFuncs{
    AddFunc: rm.addPod,           
    UpdateFunc: rm.updatePod, 
    DeleteFunc: rm.deletePod, 
}) 
rm.podStore.Indexer = podInformer.GetIndexer()
rm.podController = podInformer.GetController()
```

**pkg/controller/framework/shared_informer.go +187**

```go
func (s *sharedIndexInformer) AddEventHandler(handler ResourceEventHandler) error {
    s.startedLock.Lock()
    defer s.startedLock.Unlock()

    if s.started {
        return fmt.Errorf("informer has already started")
    }   

    listener := newProcessListener(handler)
    s.processor.listeners = append(s.processor.listeners, listener)
    return nil 
}
```

podInformer.AddEventHandler是问题的文件，这是一个eventhandler。每一种controller需要使用podinformer时，都会注册event handler，podinformer将event handler包装成listerner，然后添加到s.processor.listeners里面。

当初始化完所有的controllers，才会启动那些SharedIndexInformer

**cmd/kube-controller-manager/app/controllermanager.go +198**

```go
func StartControllers(s *options.CMServer, kubeClient *client.Client, kubeconfig *restclient.Config, stop <-chan struct{}) error {
    podInformer := informers.CreateSharedPodIndexInformer(clientset.NewForConfigOrDie(restclient.AddUserAgent(kubeconfig, "pod-informer")), ResyncPeriod(s)())
    nodeInformer := informers.CreateSharedNodeIndexInformer(clientset.NewForConfigOrDie(restclient.AddUserAgent(kubeconfig, "node-informer")), ResyncPeriod(s)())
    pvcInformer := informers.CreateSharedPVCIndexInformer(clientset.NewForConfigOrDie(restclient.AddUserAgent(kubeconfig, "pvc-informer")), ResyncPeriod(s)())
    pvInformer := informers.CreateSharedPVIndexInformer(clientset.NewForConfigOrDie(restclient.AddUserAgent(kubeconfig, "pv-informer")), ResyncPeriod(s)())
    informers := map[reflect.Type]framework.SharedIndexInformer{}
    informers[reflect.TypeOf(&api.Pod{})] = podInformer
    informers[reflect.TypeOf(&api.Node{})] = nodeInformer
    informers[reflect.TypeOf(&api.PersistentVolumeClaim{})] = pvcInformer
    informers[reflect.TypeOf(&api.PersistentVolume{})] = pvInformer
    ...
    for _, informer := range informers {
        go informer.Run(wait.NeverStop)
    }
    
    select {}
}
    
```

**pkg/controller/framework/shared_informer.go +121**

```go
func (s *sharedIndexInformer) Run(stopCh <-chan struct{}) {
    defer utilruntime.HandleCrash()

    fifo := cache.NewDeltaFIFO(cache.MetaNamespaceKeyFunc, nil, s.indexer)

    cfg := &Config{
        Queue:            fifo,
        ListerWatcher:    s.listerWatcher,
        ObjectType:       s.objectType,
        FullResyncPeriod: s.fullResyncPeriod,
        RetryOnError:     false,

        Process: s.HandleDeltas,
    }

    func() {
        s.startedLock.Lock()
        defer s.startedLock.Unlock()

        s.controller = New(cfg)
        s.started = true
    }()

    s.processor.run(stopCh)
    s.controller.Run(stopCh)
}
```

首先会构建一个controller，和前面完全一致。controller的作用就是构建一个reflector，然后将watch到的资源放入fifo这个cache里面。放入之后Process: s.HandleDeltas会对资源进行处理，看看你这个函数

**pkg/controller/framework/shared_informer.go +200**

```go
func (s *sharedIndexInformer) HandleDeltas(obj interface{}) error {
    // from oldest to newest
    for _, d := range obj.(cache.Deltas) {
        switch d.Type {
        case cache.Sync, cache.Added, cache.Updated:
            if old, exists, err := s.indexer.Get(d.Object); err == nil && exists {
                if err := s.indexer.Update(d.Object); err != nil {
                    return err 
                }   
                s.processor.distribute(updateNotification{oldObj: old, newObj: d.Object})
            } else {
                if err := s.indexer.Add(d.Object); err != nil {
                    return err 
                }   
                s.processor.distribute(addNotification{newObj: d.Object})
            }   
        case cache.Deleted:
            if err := s.indexer.Delete(d.Object); err != nil {
                return err 
            }   
            s.processor.distribute(deleteNotification{oldObj: d.Object})
        }   
    }   
    return nil 
}


func (p *sharedProcessor) distribute(obj interface{}) {
    for _, listener := range p.listeners {
        listener.add(obj)
    }
}

func (p *processorListener) add(notification interface{}) {
    p.lock.Lock()
    defer p.lock.Unlock()

    p.pendingNotifications = append(p.pendingNotifications, notification)
    p.cond.Broadcast()
}
```
哈哈，被watch的资源被传到了各个listener。那个listerner如何处理呢？再启动controller前，提前启动了s.processor.run(stopCh)，咱们再看看这个函数

**pkg/controller/framework/shared_informer.go**

```go
func (p *sharedProcessor) run(stopCh <-chan struct{}) {
    for _, listener := range p.listeners {
        go listener.run(stopCh)
        go listener.pop(stopCh)
    }
}

func (p *processorListener) pop(stopCh <-chan struct{}) {
    defer utilruntime.HandleCrash()

    for {
        blockingGet := func() (interface{}, bool) {
            p.lock.Lock()
            defer p.lock.Unlock()

            for len(p.pendingNotifications) == 0 {
                // check if we're shutdown
                select {
                case <-stopCh:
                    return nil, true
                default:
                }
                p.cond.Wait()
            }            

            nt := p.pendingNotifications[0] 
            p.pendingNotifications = p.pendingNotifications[1:]
            return nt, false          
        }

        notification, stopped := blockingGet()
        if stopped {     
            return       
        }

        select {
        case <-stopCh:   
            return
        case p.nextCh <- notification:
        }
    }
}


func (p *processorListener) run(stopCh <-chan struct{}) {
    defer utilruntime.HandleCrash() 

    for {
        var next interface{}
        select {
        case <-stopCh:
            func() {
                p.lock.Lock()
                defer p.lock.Unlock()
                p.cond.Broadcast()
            }()
            return
        case next = <-p.nextCh:
        }

        switch notification := next.(type) {
        case updateNotification:
            p.handler.OnUpdate(notification.oldObj, notification.newObj)
        case addNotification:
            p.handler.OnAdd(notification.newObj)
        case deleteNotification:
            p.handler.OnDelete(notification.oldObj)
        default:
            utilruntime.HandleError(fmt.Errorf("unrecognized notification: %#v", next))
        }
    }
}
```

listenser的add函数负责将notify装进pendingNotifications，而pop函数取出pendingNotifications的第一个nofify,输入nextCh这个channel，而run函数则负责取出notify，然后根据notify的类型(增加、删除、更新)触发相应的处理函数，这个函数是各个controller注册的。

我们看一看replication controller注册的几个函数,以add函数为例

**pkg/controller/replication/replication_controller.go +324**

```go
func (rm *ReplicationManager) addPod(obj interface{}) {
    pod := obj.(*api.Pod)

    rc := rm.getPodController(pod)
    if rc == nil {
        return
    }   
    rcKey, err := controller.KeyFunc(rc)
    if err != nil {
        glog.Errorf("Couldn't get key for replication controller %#v: %v", rc, err)
        return
    }   

    if pod.DeletionTimestamp != nil {
        // on a restart of the controller manager, it's possible a new pod shows up in a state that
        // is already pending deletion. Prevent the pod from being a creation observation.
        rm.deletePod(pod)
        return
    }   
    rm.expectations.CreationObserved(rcKey)
    rm.enqueueController(rc)
}
```

首先会根据pod返回rc，当pod不属于任何rc时，则返回。找到rc以后，更新rm.expectations.CreationObserved(rcKey)这个rc的期望值，也就是假如一个rc有4个pod，现在检测到创建了一个pod，则会将这个rc的期望值减少，变为3。然后将这个rc放入队列。

最后看看woker如何消费rm.queue

**pkg/controller/replication/replication_controller.go +542**

```go
func (rm *ReplicationManager) syncReplicationController(key string) error {
    trace := util.NewTrace("syncReplicationController: " + key)
    defer trace.LogIfLong(250 * time.Millisecond)

    startTime := time.Now()
    defer func() {
        glog.V(4).Infof("Finished syncing controller %q (%v)", key, time.Now().Sub(startTime))
    }() 

    if !rm.podStoreSynced() {
        // Sleep so we give the pod reflector goroutine a chance to run.
        time.Sleep(PodStoreSyncedPollPeriod)
        glog.Infof("Waiting for pods controller to sync, requeuing rc %v", key)
        rm.queue.Add(key)
        return nil 
    }   

    obj, exists, err := rm.rcStore.Indexer.GetByKey(key)
    if !exists {
        glog.Infof("Replication Controller has been deleted %v", key)
        rm.expectations.DeleteExpectations(key)
        return nil 
    }   
    if err != nil {
        glog.Infof("Unable to retrieve rc %v from store: %v", key, err)
        rm.queue.Add(key)
        return err
    }
    rc := *obj.(*api.ReplicationController)

    rcKey, err := controller.KeyFunc(&rc)
    if err != nil {
        glog.Errorf("Couldn't get key for replication controller %#v: %v", rc, err)
        return err
    }
    trace.Step("ReplicationController restored")
    rcNeedsSync := rm.expectations.SatisfiedExpectations(rcKey)
    trace.Step("Expectations restored")
    podList, err := rm.podStore.Pods(rc.Namespace).List(labels.Set(rc.Spec.Selector).AsSelector())
    if err != nil {
        glog.Errorf("Error getting pods for rc %q: %v", key, err)
        rm.queue.Add(key)
        return err
    }
    trace.Step("Pods listed")

    // TODO: Do this in a single pass, or use an index.
    filteredPods := controller.FilterActivePods(podList.Items)
    if rcNeedsSync {
        rm.manageReplicas(filteredPods, &rc)
    }
    trace.Step("manageReplicas done")

    fullyLabeledReplicasCount := 0
    templateLabel := labels.Set(rc.Spec.Template.Labels).AsSelector()
    for _, pod := range filteredPods {
        if templateLabel.Matches(labels.Set(pod.Labels)) {
            fullyLabeledReplicasCount++
        }
    }

    // Always updates status as pods come up or die.
    if err := updateReplicaCount(rm.kubeClient.Core().ReplicationControllers(rc.Namespace), rc, len(filteredPods), fullyLabeledReplicasCount); err != nil {
        // Multiple things could lead to this update failing. Requeuing the controller ensures
        // we retry with some fairness.
        glog.V(2).Infof("Failed to update replica count for controller %v/%v; requeuing; error: %v", rc.Namespace, rc.Name, err)
        rm.enqueueController(&rc)
    }
    return nil
}
```

## 总结

从上面的分析，我们可以看出，informer分为两类，共享和非共享。这两类informer本质上都是对reflector的封装
