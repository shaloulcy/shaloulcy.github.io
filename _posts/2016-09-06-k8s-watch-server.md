---
layout: post
author: shalou
title: "kube-apiserver的watch机制(Server端)"
category: 容器技术
tag: [k8s, kubernetes, watch]
---

*本源码分析基于k8s v1.3.6*

本分析基于k8s v1.3.6版本

什么是watch?kubelet、kube-controller-manager、kube-scheduler需要监控各种资源(pod、service等)的变化，当这些对象发生变化时(add、delete、update)，kube-apiserver能够主动通知这些组件。这个过程类似一个发布-订阅系统。

kube-apiserver的watch机制是建立在etcd的watch机制之上的，etcd的watch是没有过滤功能的，而kube-apiserver增加了过滤功能。什么是过滤？比如kubelet只对调度到本节点上的pod感兴趣，也就是pod.host=node1，而kube-scheduler只对未被调度的pod感兴趣，也就是pod.host=""，etcd只能watch到pod的add、delete、update，kube-apiserver则增加了过滤功能，将订阅方感兴趣的部分资源发给订阅方。

<!-- more -->

## kube-apiserver对etcd的操控

kube-apiserver针对每一类资源(pod、service、endpoint、replication controller、depolyments),都会与etcd建立一个连接。如：

**pkg/registry/pod/etcd/etcd.go +61**

```go
func NewStorage(opts generic.RESTOptions, k client.ConnectionInfoGetter, proxyTransport http.RoundTripper) PodStorage {
    prefix := "/pods"
    ...
    storageInterface := opts.Decorator(
        opts.Storage, cachesize.GetWatchCacheSizeByResource(cachesize.Pods), &api.Pod{}, prefix, pod.Strategy, newListFunc)

    store := &registry.Store{
        ...
        Storage: storageInterface,
    }   
    ...
    return PodStorage{
        Pod:         &REST{store, proxyTransport},
        ...
    }
}
```

这里针对pod资源，建立了于etcd的连接，具体针对etcd的操作封装在PodStorage.Pod.Storage里面，StorageInterface=opts.Decorator(
        opts.Storage, cachesize.GetWatchCacheSizeByResource(cachesize.Pods), &api.Pod{}, prefix, pod.Strategy, newListFunc),Decorator是什么函数呢？我们必须先弄清opts是什么,opts的类型为generic.RESTOptions，它在之前初始化了。
        
**pkg/master/master.go +476**
     
```go
    restOptions := func(resource string) generic.RESTOptions {
        return m.GetRESTOptionsOrDie(c, api.Resource(resource))
    }
    ...
    podStorage := podetcd.NewStorage(
        restOptions("pods"),
        kubeletclient.ConnectionInfoGetter(nodeStorage.Node),
        m.ProxyTransport,
    )
```

又上面的函数可以知道是由GetRESTOptionsOrDie函数生成了opts，从下面的函数可知道，opts结构体包含了三部分：Storage、Decorator和DeleteCollectionWorkers,Storage是实际发生存储的接口，而Decorator从字面意思，其实是针对storage的一个装饰器。这个装饰器对于kube-apiserver具有重要作用，可以说是watch的基础吧，我们会在后面进行介绍。

**pkg/master/master.go +824**

```go
func (m *Master) GetRESTOptionsOrDie(c *Config, resource unversioned.GroupResource) generic.RESTOptions {
    storage, err := c.StorageFactory.New(resource)
    if err != nil {
        glog.Fatalf("Unable to find storage destination for %v, due to %v", resource, err.Error())
    }
    
    return generic.RESTOptions{
        Storage:                 storage,
        Decorator:               m.StorageDecorator(),
        DeleteCollectionWorkers: m.deleteCollectionWorkers,
    }
}  
```
我们首先来看看Storage是怎么生成的。它是通过storage, err := c.StorageFactory.New(resource)生成的。StorageFactory是个针对etcd的工厂函数，任何一种资源的etcd都是由它创建的，我们这里不介绍它的生成。New函数位于

**pkg/genericapiserver/storage_factory.go +158**

```go
func (s *DefaultStorageFactory) New(groupResource unversioned.GroupResource) (storage.Interface, error) {
    ...
    etcdPrefix := s.StorageConfig.Prefix
    ...
    // operate on copy
    config := s.StorageConfig
    config.Prefix = etcdPrefix
    ...
    return s.newStorageFn(config, codec)
}


func newStorage(config storagebackend.Config, codec runtime.Codec) (etcdStorage storage.Interface, err error) {
    return storagebackendfactory.Create(config, codec)
}
```
我们省略了大部分代码，这些代码主要完成资源的序列化、反序列化、版本转换等功能的，这些功能我们暂时不关注。etcPrefix默认为/registry，函数传进来的参数groupResource是pods,相应的在etcd中存储路径为/registry/pods，newStorageFn指的就是newStorage这个函数，这个函数调用storagebackendfactory.Create函数，storagebackendfactory又是一个工厂类，目前k8s的后端存储支持etcd2和etcd3。这个函数位于

**pkg/storage/storagebackend/factory/factory.go +28**

```go
func Create(c storagebackend.Config, codec runtime.Codec) (storage.Interface, error) {
    switch c.Type {
    case storagebackend.StorageTypeUnset, storagebackend.StorageTypeETCD2:
        return newETCD2Storage(c, codec)
    case storagebackend.StorageTypeETCD3:
        return newETCD3Storage(c, codec)
    default: 
        return nil, fmt.Errorf("unknown storage type: %s", c.Type)
    }
} 
```

默认的存储后端为etcd2,我们看看函数newETCD2Storage,函数位于

**pkg/storage/storagebackend/factory/etcd2.go +34**

```go
func newETCD2Storage(c storagebackend.Config, codec runtime.Codec) (storage.Interface, error) {
    tr, err := newTransportForETCD2(c.CertFile, c.KeyFile, c.CAFile)
    if err != nil {
        return nil, err 
    }   
    client, err := newETCD2Client(tr, c.ServerList)
    if err != nil {
        return nil, err 
    }   
    return etcd.NewEtcdStorage(client, codec, c.Prefix, c.Quorum, c.DeserializationCacheSize), nil 
}
```
函数首先生成一个利用etcd server的地址生成一个etcdclient，借助函数etcd.NewEtcdStorage生成storage，这个函数位于：

**pkg/storage/etcd/etcd_helper.go +45**

```go
func NewEtcdStorage(client etcd.Client, codec runtime.Codec, prefix string, quorum bool, cacheSize int) storage.Interface {
    return &etcdHelper{
        etcdMembersAPI: etcd.NewMembersAPI(client),
        etcdKeysAPI:    etcd.NewKeysAPI(client),
        codec:          codec,
        versioner:      APIObjectVersioner{},
        copier:         api.Scheme,
        pathPrefix:     path.Join("/", prefix),
        quorum:         quorum,
        cache:          utilcache.NewCache(cacheSize),
    }   
}
```
最后生成etcdHelper结构，etcdHelper实现了storage.Interface结构，kube-apiserver的所有操作最后都会通过这个结构体持久化到etcd中。

## opts.Decorator的背后

从前面我们可以知道，最终会生成一个针对etcd的Storage，然后会使用一个装饰器函数去包装Storage，opts.Decorator(opts.Storage, cachesize.GetWatchCacheSizeByResource(cachesize.Pods), &api.Pod{}, prefix, pod.Strategy, newListFunc)。Decorator到底是什么函数呢，其代码位于

**pkg/genericapiserver/genericapiserver.go +256**

```go
func (s *GenericAPIServer) StorageDecorator() generic.StorageDecorator {
    if s.enableWatchCache {
        return registry.StorageWithCacher
    }
    return generic.UndecoratedStorage
} 
```
假如enableWatchCache为真，返回registry.StorageWithCacher，为假则返回generic.UndecoratedStorage，enableWatchCache默认为真，因此Decorator就是StorageWithCacher，即这是一个带有cache的storage，函数代码位于

**pkg/registry/generic/registry/storage_factory.go +27**

```go
func StorageWithCacher(
    storageInterface storage.Interface,
    capacity int,
    objectType runtime.Object,
    resourcePrefix string,
    scopeStrategy rest.NamespaceScopedStrategy,
    newListFunc func() runtime.Object) storage.Interface {
    return storage.NewCacher(
        storageInterface, capacity, etcdstorage.APIObjectVersioner{},
        objectType, resourcePrefix, scopeStrategy, newListFunc)
}
```
调用函数storage.NewCacher，这里的storageinterface就是前面生成的etcd的接口。NewCacher函数接着会调用NewCacherFromConfig(config),我们看看这个函数，

**pkg/storage/cacher.go +147**

```go
func NewCacherFromConfig(config CacherConfig) *Cacher {
    watchCache := newWatchCache(config.CacheCapacity)
    listerWatcher := newCacherListerWatcher(config.Storage, config.ResourcePrefix, .NewListFunc)
    ...

    cacher := &Cacher{
        usable:     sync.RWMutex{},
        storage:    config.Storage,
        watchCache: watchCache,
        reflector:  cache.NewReflector(listerWatcher, config.Type, watchCache, 0),
        watcherIdx: 0,
        watchers:   make(map[int]*cacheWatcher),
        versioner:  config.Versioner,
        keyFunc:    config.KeyFunc,
        ...
    }
    cacher.usable.Lock()
    watchCache.SetOnEvent(cacher.processEvent)
    go func() {
        defer cacher.stopWg.Done()
        wait.Until(
            func() {
                if !cacher.isStopped() {
                    cacher.startCaching(stopCh)
                }
            }, time.Second, stopCh,
        )
    }()
    return cacher
}
```
从函数我们看出，最后返回了一个cacher,这个cacher有几个重要的成员函数：storage、watchCache、watchers，reflector，最后还启动了一个线程做cacher.startCaching工作。storage就是前面生成的etcd的接口,watchCache是一个cache，用来存储apiserver从etcd那里watch到的对象，watchers是一个map，map的值类型为cacheWatcher，这个对象比较有意思，当kubelet、kube-scheduler需要watch某类资源时，他们会向kube-apiserver发起watch请求，kube-apiserver就会生成一个cacheWatcher，他们负责将watch的资源通过http从apiserver传递到kubelet、kube-scheduler。reflector这个对象，包含两个重要的数据成员listerWatcher和watchCache,而listerWatcher包装了config.Storage，他的工作主要是将watch到的对象存到watcherCache中。

我们看看watcherCacher类型

```go
type watchCache struct {

    // Maximum size of history window.
    capacity int

    cache      []watchCacheElement
    startIndex int 
    endIndex   int 

    store cache.Store
    onEvent func(watchCacheEvent)
}
```

他包含两个重要成员，cache和store。cache存储的是事件(add、delete、update)，store则存储资源对象。reflector是如何将apiserver针对etcd的watch结果存到watchCache中去的了，主要通过前面提到的函数，通过线程启动的startCaching函数

**pkg/storage/cacher.go +196**

```go
func (c *Cacher) startCaching(stopChannel <-chan struct{}) {
    successfulList := false
    c.watchCache.SetOnReplace(func() {
        successfulList = true
        c.usable.Unlock()
    })  
    defer func() {
        if successfulList {
            c.usable.Lock()
        }   
    }() 

    c.terminateAllWatchers()

    if err := c.reflector.ListAndWatch(stopChannel); err != nil {
        glog.Errorf("unexpected ListAndWatch error: %v", err)
    }   
}
```
他首先会通过terminateAllWatchers注销所有的cachewatcher,因为这个时候apiserver还处于初始化阶段，因此不可能接受其他组件的watch，也就不可能有watcher。然后调用c.reflector.ListAndWatch函数，前面我们说过reflector主要将apiserver自己watch到的资源存储到watchCache中。看看该函数代码

**pkg/client/cache/reflector.go +252**

```go
func (r *Reflector) ListAndWatch(stopCh <-chan struct{}) error {
    ...
    list, err := r.listerWatcher.List(options)
    ...
    items, err := meta.ExtractList(list)
    ...
    if err := r.syncWith(items, resourceVersion); err != nil {
        return fmt.Errorf("%s: Unable to sync list result: %v", r.name, err)
    }
    ...
    go func() {
        for {
            select {
            case <-resyncCh:
            case <-stopCh:
                return
            }
            ...
            if err := r.store.Resync(); err != nil {
                resyncerrc <- err
                return
            }
            cleanup()
            resyncCh, cleanup = r.resyncChan()
        }
       }()

    for {
        ...
        w, err := r.listerWatcher.Watch(options)
        

        if err := r.watchHandler(w, &resourceVersion, resyncerrc, stopCh); err != nil {
            ...
            return nil
        }
    }
}
```

函数首先调用r.listerWatcher.List(options),如下所示，它回去调用lw.storage.List，而storage就是etcd,因此最终回去调用etcd的List函数，即把所有的pods都返回。

**pkg/storage/cacher.go +450**

```go
func (lw *cacherListerWatcher) List(options api.ListOptions) (runtime.Object, error) {
    list := lw.newListFunc()
    if err := lw.storage.List(context.TODO(), lw.resourcePrefix, "", Everything, list); err != nil {
        return nil, err
    }
    return list, nil
}
```
然后调用r.syncWith(items, resourceVersion),这个函数背后调用r.store.Replace(found, resourceVersion),将所有的pods添加到watchCache中去，然后启动线程r.store.Resync，但这个函数为空，不起作用。

函数接下来调用r.listerWatcher.Watch(options)，真正的watch。调用如下

```go
func (lw *cacherListerWatcher) Watch(options api.ListOptions) (watch.Interface, error) {
    return lw.storage.WatchList(context.TODO(), lw.resourcePrefix, options.ResourceVersion, Everything)
}
```

从函数我们可以知道，它最终去调用etcd的WatchList函数，看看该函数

**pkg/storage/etcd/etcd_helper.go +241**

```go
func (h *etcdHelper) WatchList(ctx context.Context, key string, resourceVersion string, filter storage.FilterFunc) (watch.Interface, error) {
    watchRV, err := storage.ParseWatchResourceVersion(resourceVersion)
    key = h.prefixEtcdKey(key)
    w := newEtcdWatcher(true, h.quorum, exceptKey(key), filter, h.codec, h.versioner, nil, h)
    go w.etcdWatch(ctx, h.etcdKeysAPI, key, watchRV)
    return w, nil        
}
```

函数会生成一个etcdWatcher,生成函数为newEtcdWatcher(true, h.quorum, exceptKey(key), filter, h.codec, h.versioner, nil, h)。

**pkg/storage/etcd/etcd_watcher.go +116**

```go
func newEtcdWatcher(
    list bool, quorum bool, include includeFunc, filter storage.FilterFunc,
    encoding runtime.Codec, versioner storage.Versioner, transform TransformFunc,
    cache etcdCache) *etcdWatcher {
    w := &etcdWatcher{
        encoding:  encoding,
        versioner: versioner,
        transform: transform,
        list:      list,
        quorum:    quorum,
        include:   include,
        filter:    filter,
        
        etcdIncoming: make(chan *etcd.Response, 100),
        etcdError:    make(chan error, 1),
        outgoing:     make(chan watch.Event),
        userStop:     make(chan struct{}),
        stopped:      false,
        wg:           sync.WaitGroup{},
        cache:        cache,
        ctx:          nil,
        cancel:       nil,
    }
    w.emit = func(e watch.Event) {
        select {
        case w.outgoing <- e:
        case <-w.userStop:
        }
    }
    go w.translate()
    return w
}
```

该函数生成一个etcdWatch，然后有两个比较重要的channel：etcdIncoming和outgoging。最后这个函数会启动一个线程运行w.translate()。这个会在后面用到


回到前面源代码，生成etcdWatcher以后，创建一个线程运行etcdWatch，继续查看etcdWatch函数

**pkg/storage/etcd/etcd_watcher.go +167**

```go
func (w *etcdWatcher) etcdWatch(ctx context.Context, client etcd.KeysAPI, key string, resourceVersion uint64) {
    var watcher etcd.Watcher
    returned := func() bool {
        if w.stopped {
            // Watcher has already been stopped - don't event initiate it here.
            return true
        }   
        w.wg.Add(1)
        done = w.wg.Done
        if resourceVersion == 0 { 
            latest, err := etcdGetInitialWatchState(ctx, client, key, w.list, w.quorum, w.etcdIncoming)
            if err != nil {
                w.etcdError <- err 
                return true
            }   
            resourceVersion = latest
        }   

        opts := etcd.WatcherOptions{
            Recursive:  w.list,
            AfterIndex: resourceVersion,
        }   
        watcher = client.Watcher(key, &opts)
        w.ctx, w.cancel = context.WithCancel(ctx)
        return false
    }()
    defer done()
    if returned {
        return
    }

    for {
        resp, err := watcher.Next(w.ctx)
        if err != nil {
            w.etcdError <- err
            return
        }
        w.etcdIncoming <- resp
    }
}
```
首先当resourceVersion=0时，也就是第一次watch的时候，会调用etcdGetInitialWatchState，接下来则会进入一个for循环不停的调用watcher.Next(w.ctx)，并把结果输入到w.etcdIncoming这个channel中.etcdGetInitialWatchState是get所有的pods，然后将结果输入到etcIncoming中。即我们把第一次get的结果和随后watch的结果都输入到了etcdIncoming这个channel，谁将从这个channel获取结果呢？答案是前面说到的w.translate()函数，我们看看这个函数

**pkg/storage/etcd/etcd_watcher.go +269**

```go
func (w *etcdWatcher) translate() {
    defer w.wg.Done()
    defer close(w.outgoing)
    defer utilruntime.HandleCrash()

    for {
        select {
        ....
        case res, ok := <-w.etcdIncoming:
            if ok {
                ...
                w.sendResult(res)
            }
        }
    }
}

func (w *etcdWatcher) sendResult(res *etcd.Response) {
    switch res.Action {
    case EtcdCreate, EtcdGet:
        w.sendAdd(res)
    case EtcdSet, EtcdCAS:
        w.sendModify(res)
    case EtcdDelete, EtcdExpire, EtcdCAD:
        w.sendDelete(res)
    default:    
        utilruntime.HandleError(fmt.Errorf("unknown action: %v", res.Action))
    }               
}

func (w *etcdWatcher) sendAdd(res *etcd.Response) {
    obj, err := w.decodeObject(res.Node)
    if !w.filter(obj) {
        return
    }
    action := watch.Added
    if res.Node.ModifiedIndex != res.Node.CreatedIndex {
        action = watch.Modified
    }
    w.emit(watch.Event{
        Type:   action,
        Object: obj,
    })
}

w.emit = func(e watch.Event) {
    select {
    case w.outgoing <- e:
    case <-w.userStop:
    }
}

```

这个函数从etcIncoming获取到首次get和随后watch到的对象，然后调用w.SendResult函数发送结果，Create、Get的结果调用sendAdd函数发送，Set结果调用sendModify函数发送,Delete、Expore结果调用sendDelete函数。看看sendAdd函数，他首先将结果反序列化为资源对象，然后组装成一个event对象，然后发送这个event对象到outgoing这个channel上。谁负责接收这个管道上的内容呢？前面提到过outgoing和etcdIncoming都被包装在etcdWatcher里面。

回到函数func (r *Reflector) ListAndWatch(stopCh <-chan struct{}) ( **pkg/client/cache/reflector.go +252** )，该函数将生成的etcdWatcher传入函数r.watchHandler(w, &resourceVersion, resyncerrc, stopCh)，我们看看该函数的实现

**pkg/client/cache/reflector.go +351**

```go
func (r *Reflector) watchHandler(w watch.Interface, resourceVersion *string, errc chan error, stopCh <-chan struct{}) error {
...
loop:
    for {
        select {
        ...
        case event, ok := <-w.ResultChan():
            ...
            switch event.Type {
            case watch.Added:
                r.store.Add(event.Object)
            case watch.Modified:
                r.store.Update(event.Object)
            case watch.Deleted:
                r.store.Delete(event.Object)
            default:
                utilruntime.HandleError(fmt.Errorf("%s: unable to understand watch event %#v", r.name, event))
            }
            *resourceVersion = newResourceVersion
            r.setLastSyncResourceVersion(newResourceVersion)
            eventCount++
        }
    }
    ...
}
```
函数中的w.ResultChan()指的就是outgoing这个channel，该函数从outging将event对象从outgoing中读出来，然后根据event.Type去操作r.store，即操作watchCache。操作如下

**pkg/storage/watch_cache.go +116**

```go
func (w *watchCache) Add(obj interface{}) error {
    object, resourceVersion, err := objectToVersionedRuntimeObject(obj)
    if err != nil {      
        return err       
    }
    event := watch.Event{Type: watch.Added, Object: object}

    f := func(obj runtime.Object) error { return w.store.Add(obj) } 
    return w.processEvent(event, resourceVersion, f)
}

func (w *watchCache) Update(obj interface{}) error {
    object, resourceVersion, err := objectToVersionedRuntimeObject(obj)
    if err != nil {      
        return err       
    }  
    event := watch.Event{Type: watch.Modified, Object: object}

    f := func(obj runtime.Object) error { return w.store.Update(obj) }
    return w.processEvent(event, resourceVersion, f)
}

func (w *watchCache) Delete(obj interface{}) error {
    object, resourceVersion, err := objectToVersionedRuntimeObject(obj)
    if err != nil {      
        return err
    }  
    event := watch.Event{Type: watch.Deleted, Object: object}

    f := func(obj runtime.Object) error { return w.store.Delete(obj) }
    return w.processEvent(event, resourceVersion, f)
}
```

这些函数将object重新包装成event，然后调用w.processEvent函数处理

**pkg/storage/watch_cache.go +172**

```go
func (w *watchCache) processEvent(event watch.Event, resourceVersion uint64, updateFunc func(runtime.Object) error) error {
    previous, exists, err := w.store.Get(event.Object)
    if err != nil {
        return err 
    }   
    var prevObject runtime.Object
    if exists {
        prevObject = previous.(runtime.Object)
    }   
    watchCacheEvent := watchCacheEvent{event.Type, event.Object, prevObject, resourceVersion}
    if w.onEvent != nil {
        w.onEvent(watchCacheEvent)
    }   
    w.updateCache(resourceVersion, watchCacheEvent)
    w.resourceVersion = resourceVersion
    w.cond.Broadcast()
    return updateFunc(event.Object)
}
```

该函数将event包装成watchCacheEvent，假如w.onEvent不为空，就调用w.onEvent(watchCacheEvent)，然后滴啊用updateCache更新cache，cache的是对象，调用updateFunc更新store。onEvent是什么函数呢？我们前面在生成cacher对象时，调用了watchCache.SetOnEvent(cacher.processEvent)(**pkg/storage/cacher.go +179**)将onEvent设置为processEvent，注意前面的processEvent是watchcache的，而这里的processEvent是cacher的，watchcache是cacher的一个成员变量。

**pkg/storage/cacher.go +368**

```go
func (c *Cacher) processEvent(event watchCacheEvent) {
    c.Lock()
    defer c.Unlock()
    for _, watcher := range c.watchers {
        watcher.add(event)
    }
}
```

从函数我们可以看到，它将event添加到各个watcher里面。watcher是如何生成的呢？这是我们下一节要讲的内容。


从上面的分析我们可以看到，当kube-apiserver进行初始化时，kube-apiserver回去连接etcd，针对每一类资源生成一个Storage，然后将Storage包装成cacher，cacher会建立对etcd的watch，并将watch的结果存入watchCache,watchCache会将event存入cache，将object存入store,同时还会将event也添加到watchers。watcher是kube-apiserver watch的发布方和订阅方的枢纽。


## watchers的作用

kube-apiserver的watch功能是作为一个restful api提供给其他组件(kubelet、kube-controller-manager、kube-scheduler、kube-proxy)。因此watch的处理流程和PUT、DELETE、GET等api处理流程类似。这部分代码位于

**pkg/apiserver/api_installer.go +669**

```go
        case "WATCHLIST": // Watch all resources of a kind.
            doc := "watch individual changes to a list of " + kind
            if hasSubresource {
                doc = "watch individual changes to a list of " + subresource + " of " + kind
            }   
            handler := metrics.InstrumentRouteFunc(action.Verb, resource, ListResource(lister, watcher, reqScope, true, a.minRequestTimeout))
            route := ws.GET(action.Path).To(handler).
                Doc(doc).
                Param(ws.QueryParameter("pretty", "If 'true', then the output is pretty printed.")).
                Operation("watch"+namespaced+kind+strings.Title(subresource)+"List").
                Produces(a.group.Serializer.SupportedStreamingMediaTypes()...).
                Returns(http.StatusOK, "OK", versionedWatchEvent).
                Writes(versionedWatchEvent)
            if err := addObjectParams(ws, route, versionedListOptions); err != nil {
                return nil, err 
            }   
            addParams(route, action.Params)
            ws.Route(route)
```

查看函数ListResource(lister, watcher, reqScope, true, a.minRequestTimeout)，


**pkg/apiserver/resthandler.go  +234**

```go
func ListResource(r rest.Lister, rw rest.Watcher, scope RequestScope, forceWatch bool, minRequestTimeout time.Duration) restful.RouteFunction {
    return func(req *restful.Request, res *restful.Response) {
            
        opts := api.ListOptions{}
        if err := scope.ParameterCodec.DecodeParameters(req.Request.URL.Query(), scope.Kind.GroupVersion(), &opts); err != nil {
            scope.err(err, res.ResponseWriter, req.Request)
            return
        }
        ...
        if (opts.Watch || forceWatch) && rw != nil {
            watcher, err := rw.Watch(ctx, &opts)
            ...
            serveWatch(watcher, scope, req, res, timeout)
            return
        }
        ...
    }
}
```
该函数首先调用rw.Watch(ctx, &opts)

**pkg/registry/generic/registry/store.go +779**


```go
func (e *Store) Watch(ctx api.Context, options *api.ListOptions) (watch.Interface, error) {
    label := labels.Everything()
    if options != nil && options.LabelSelector != nil {
        label = options.LabelSelector
    }                    
    field := fields.Everything()
    if options != nil && options.FieldSelector != nil {
        field = options.FieldSelector
    }
    resourceVersion := ""
    if options != nil {  
        resourceVersion = options.ResourceVersion 
    }
    return e.WatchPredicate(ctx, e.PredicateFunc(label, field), resourceVersion)
}

// WatchPredicate starts a watch for the items that m matches.
func (e *Store) WatchPredicate(ctx api.Context, m generic.Matcher, resourceVersion string) (watch.Interface, error) {
    filterFunc := e.filterAndDecorateFunction(m)

    if name, ok := m.MatchesSingle(); ok {
        if key, err := e.KeyFunc(ctx, name); err == nil {
            if err != nil {           
                return nil, err           
            }
            return e.Storage.Watch(ctx, key, resourceVersion, filterFunc)
        }
        // if we cannot extract a key based on the current context, the optimization is skipped
    }  

    return e.Storage.WatchList(ctx, e.KeyRootFunc(ctx), resourceVersion, filterFunc)
}
```

然后调用

**pkg/storage/cacher.go +245**

```go
func (c *Cacher) Watch(ctx context.Context, key string, resourceVersion string, filter FilterFunc) (watch.Interface, error) {
        initEvents, err := c.watchCache.GetAllEventsSinceThreadUnsafe(watchRV)
    if err != nil {
        return newErrWatcher(err), nil 
    }   

    watcher := newCacheWatcher(watchRV, initEvents, filterFunction(key, c.keyFunc, filter), forgetWatcher(c, c.watcherIdx))
    c.watchers[c.watcherIdx] = watcher
    c.watcherIdx++
    return watcher, nil 
}

// Implements storage.Interface.
func (c *Cacher) WatchList(ctx context.Context, key string, resourceVersion string, filter FilterFunc) (watch.Interface, error) {
    return c.Watch(ctx, key, resourceVersion, filter)
}
```

从这里我们可以看到传入了一个filter，前面提供apiserver的watch是带过滤功能的，就是有这个filter实现的，然后调用newCacheWatcher生成一个watcher，并将这个watcher插入到cacher.watchers中去，这里的watcher便是上一节中最后的watcher，上一节中我们提到过watchCache会将event通过add添加到watcher中。我们先看看watcher.add函数

**pkg/storage/cacher.go +547**

```go
func (c *cacheWatcher) add(event watchCacheEvent) {
    // Try to send the event immediately, without blocking.
    select {             
    case c.input <- event:    
        return           
    default:             
    }
    ... 
}
```
从函数可以看出，会将event输出到watcher.input这个channel中，谁去消费这个channel中。我们去看看创建这个watcher的函数newCacheWatcher

**pkg/storage/cacher.go +513**

```go
func newCacheWatcher(resourceVersion uint64, initEvents []watchCacheEvent, filter FilterFunc, forget func(bool)) *cacheWatcher {
    watcher := &cacheWatcher{
        input:   make(chan watchCacheEvent, 10),
        result:  make(chan watch.Event, 10),
        filter:  filter,
        stopped: false,
        forget:  forget,
    }   
    go watcher.process(initEvents, resourceVersion)
    return watcher
}

func (c *cacheWatcher) process(initEvents []watchCacheEvent, resourceVersion uint64) {
    for _, event := range initEvents {
        c.sendWatchCacheEvent(event)
    }   
    defer close(c.result)
    defer c.Stop()
    for {
        event, ok := <-c.input
        if !ok {
            return
        }
        // only send events newer than resourceVersion
        if event.ResourceVersion > resourceVersion {
            c.sendWatchCacheEvent(event)
        }
    }
}

func (c *cacheWatcher) sendWatchCacheEvent(event watchCacheEvent) {
    curObjPasses := event.Type != watch.Deleted && c.filter(event.Object)
    oldObjPasses := false
    if event.PrevObject != nil {
        oldObjPasses = c.filter(event.PrevObject)
    }
    if !curObjPasses && !oldObjPasses {
        // Watcher is not interested in that object.
        return
    }

    object, err := api.Scheme.Copy(event.Object)
    if err != nil {
        glog.Errorf("unexpected copy error: %v", err)
        return
    }
    switch {
    case curObjPasses && !oldObjPasses:
        c.result <- watch.Event{Type: watch.Added, Object: object}
    case curObjPasses && oldObjPasses:
        c.result <- watch.Event{Type: watch.Modified, Object: object}
    case !curObjPasses && oldObjPasses:
        c.result <- watch.Event{Type: watch.Deleted, Object: object}
    }

```
从函数可以看到创建watcher后，立即创建了一个线程执行watcher.process。process首先会调用 sendWatchCacheEvent发送所有的initEvents，然后for循环从管道input接收event，这些event就是kube-apiserver watch etcd的结果。

 sendWatchCacheEvent对调用c.filter函数对watch的结果进行过滤，然后将过滤后的结果包装成event，发送到c.result这个channel。
 
 
 回到本节开头，创建好watcher以后，函数会调用serveWatch(watcher, scope, req, res, timeout)。我们看看
 
 **pkg/apiserver/watch.go +64**

```go
func serveWatch(watcher watch.Interface, scope RequestScope, req *restful.Request, res *restful.Response, timeout time.Duration) {
    ...
    server := &WatchServer{
        watching: watcher,
        scope:    scope,

        useTextFraming:  useTextFraming,
        mediaType:       serializer.MediaType,
        framer:          serializer.Framer,
        encoder:         encoder,
        embeddedEncoder: embeddedEncoder,
        fixup: func(obj runtime.Object) {
            if err := setSelfLink(obj, req, scope.Namer); err != nil {
                utilruntime.HandleError(fmt.Errorf("failed to set link for object %v: %v", reflect.TypeOf(obj), err))
            }   
        },  

        t: &realTimeoutFactory{timeout},
    }   

    server.ServeHTTP(res.ResponseWriter, req.Request)
}
```

**pkg/apiserver/watch.go +**

```go
func (s *WatchServer) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    ...
    ch := s.watching.ResultChan()
    for {
        select {
        case <-cn.CloseNotify():
            return
        case <-timeoutCh:
            return
        case event, ok := <-ch:
            ...
            obj := event.Object
            s.fixup(obj)
            if err := s.embeddedEncoder.Encode(obj, buf); err != nil {
                ....
            }

            unknown.Raw = buf.Bytes()
            event.Object = &unknown
            
            *internalEvent = versioned.InternalEvent(event)
            if err := e.Encode(internalEvent); err != nil {
                utilruntime.HandleError(fmt.Errorf("unable to encode watch object: %v (%#v)", err, e))
                // client disconnect.
                return
            }
            if len(ch) == 0 {
                flusher.Flush()
            }

            buf.Reset()
        }
    }
}
```

从上述两段代码可以看出，首先利用返回的watcher创建一个WatchServer,WatchServer开始运行，ch := s.watching.ResultChan(),ch就是前面说到的channel result，从result取出event，然后将其序列化，最后通过发送出去。这样就讲watch的结果发送给订阅方了

## 总结

kube-apiserver的watch机制如下：kube-apiserver初始化时，建立对etcd的连接，并对etcd进行watch，将watch的结果存入watchCache。当其他组件需要watch资源时，其他组件向apiserver发送一个watch请求，这个请求是可以带filter函数的，apiserver针对这个请求会创建一个watcher，并基于watcher创建WatchServer，watchCache watch的对象，首先会通过filter函数的过滤，假如过滤通过的话，则会通过WatcherServer发送给订阅组件。
