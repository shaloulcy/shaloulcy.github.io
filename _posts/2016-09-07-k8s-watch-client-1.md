---
layout: post
author: shalou
title:  "kube-apiserver的watch机制(Client端)(第1部分)"
category: 容器技术
tag: [k8s, kubernetes]
---

*本源码分析基于k8s v1.3.6*

上一章中，我们介绍了kube-apiserver的watch机制server端，这一篇幅中我们将介绍watch机制的client端，它是如何发起watch，并接收watch到的对象。这里的客户端包括kubelet、kube-scheduler、kube-controller-manager以及kube-proxy。我们以kubelet为例

## kubelet对pod资源的watch

kubelet主要的工作就是创建、维护、销毁pod，因此他需要对pod进行watch，而且它watch想要watch的pod是那些已经分配到本节点的pods

<!-- more -->

**./cmd/kubelet/app/server.go +891**

```go
func CreateAndInitKubelet(kc *KubeletConfig) (k KubeletBootstrap, pc *config.PodConfig, err error) {
    var kubeClient clientset.Interface
    if kc.KubeClient != nil {
        kubeClient = kc.KubeClient
    }    
    ...
    pc = kc.PodConfig
    if pc == nil {
        pc = makePodSourceConfig(kc)
    }
    ...
}

func makePodSourceConfig(kc *KubeletConfig) *config.PodConfig {
    // source of all configuration
    cfg := config.NewPodConfig(config.PodConfigNotificationIncremental, kc.Recorder)

    // define file config source
    if kc.ConfigFile != "" { 
        glog.Infof("Adding manifest file: %v", kc.ConfigFile)
        config.NewSourceFile(kc.ConfigFile, kc.NodeName, kc.FileCheckFrequency, cfg.Channel(kubetypes.FileSource))
    }    

    // define url config source
    if kc.ManifestURL != "" { 
        glog.Infof("Adding manifest url %q with HTTP header %v", kc.ManifestURL, kc.ManifestURLHeader)
        config.NewSourceURL(kc.ManifestURL, kc.ManifestURLHeader, kc.NodeName, kc.HTTPCheckFrequency, cfg.Channel(kubetypes.HTTPSource))
    }    
    if kc.KubeClient != nil {
        glog.Infof("Watching apiserver")
        config.NewSourceApiserver(kc.KubeClient, kc.NodeName, cfg.Channel(kubetypes.ApiserverSource))
    }    
    return cfg
}
```

从makePodSourceConfig可以看出，pod的source有三种，第一种是通过文件获得，文件一般放在/etc/kubernetes/manifests目录下面，启动kubelet时可以通过指定--config覆盖，第二种也是通过文件过得，只不过文件是通过URL获取的,URL可以在启动kubelet时通过ManifestURL指定，第三种是通过kube-apiserver获取。其中前两种模式下，我们称kubelet运行在standalone模式下。现在我们对第三种方式比较感兴趣。

**pkg/kubelet/config/apiserver.go +29**

```go
func NewSourceApiserver(c *clientset.Clientset, nodeName string, updates chan<- interface{}) {
    lw := cache.NewListWatchFromClient(c.CoreClient, "pods", api.NamespaceAll, fields.OneTermEqualSelector(api.PodHostField, nodeName))
    newSourceApiserverFromLW(lw, updates)
}
```

cache.NewListWatchFromClient似曾相识，它的第一个参数c.CoreClient，这个client就是apiserver的client,“Pods”指需要watch的资源是pod，api.NamespaceAll指对所有namespaces下的pod都敢兴趣，最后一个参数比较有趣，其实是一个过滤函数，即只对api.PodHostField=nodeName的pod感兴趣，也就是只对scheduler到本节点上的Pod感兴趣。我们深入看看这个函数

**pkg/client/cache/listwatch.go +49**

```go
func NewListWatchFromClient(c Getter, resource string, namespace string, fieldSelector fields.Selector) *ListWatch {
    listFunc := func(options api.ListOptions) (runtime.Object, error) {
        return c.Get().
            Namespace(namespace).
            Resource(resource).
            VersionedParams(&options, api.ParameterCodec).
            FieldsSelectorParam(fieldSelector).
            Do().
            Get()
    }   
    watchFunc := func(options api.ListOptions) (watch.Interface, error) {
        return c.Get().
            Prefix("watch").
            Namespace(namespace).
            Resource(resource).
            VersionedParams(&options, api.ParameterCodec).
            FieldsSelectorParam(fieldSelector).
            Watch()
    }   
    return &ListWatch{ListFunc: listFunc, WatchFunc: watchFunc}
}
```
函数返回的是一个ListWatch结构体，里面包含listFunc和watchFunc，其中listFunc用于List,而watchFunc用于Watch。返回的ListWatch会传入newSourceApiserverFromLW这个函数，让我们看看这个函数干了一些什么事情。

**pkg/kubelet/config/apiserver.go +35**

```go
func newSourceApiserverFromLW(lw cache.ListerWatcher, updates chan<- interface{}) {
    send := func(objs []interface{}) {
        var pods []*api.Pod
        for _, o := range objs {
            pods = append(pods, o.(*api.Pod))
        }   
        updates <- kubetypes.PodUpdate{Pods: pods, Op: kubetypes.SET, Source: kubetypes.ApiserverSource}
    }   
    cache.NewReflector(lw, &api.Pod{}, cache.NewUndeltaStore(send, cache.MetaNamespaceKeyFunc), 0).Run()
}
```

从函数可以看出，首先创建了一个send函数，从函数的内容可以看出，它是将从apiserver获取的pods传送到updates这个管道里面。这些Pods是从哪里获取的了，可以看看cache.NewReflector(lw, &api.Pod{}, cache.NewUndeltaStore(send, cache.MetaNamespaceKeyFunc), 0).Run()这个函数，首先构建了一个reflector，然后run这个reflector。reflector有两个重要成员，一个是ListWatch，主要用来对apiserver进行watch，另外一个是cache，用来存储watch到的对象。我们先看看Run(）函数

**pkg/client/cache/reflector.go +201**

```go
func (r *Reflector) Run() {
    glog.V(3).Infof("Starting reflector %v (%s) from %s", r.expectedType, r.resyncPeriod, r.name)
    go wait.Until(func() {
        if err := r.ListAndWatch(wait.NeverStop); err != nil {
            utilruntime.HandleError(err)
        }   
    }, r.period, wait.NeverStop)
}
```

函数启动一个线程，执行reflector的ListAndWatch函数，我们看看这个函数。

**pkg/client/cache/reflector.go +252**

```go
func (r *Reflector) ListAndWatch(stopCh <-chan struct{}) error {
    ...
    options := api.ListOptions{ResourceVersion: "0"}
    list, err := r.listerWatcher.List(options)
    ...  
    resourceVersion = listMetaInterface.GetResourceVersion()
    items, err := meta.ExtractList(list)
    ... 
    if err := r.syncWith(items, resourceVersion); err != nil {
        return fmt.Errorf("%s: Unable to sync list result: %v", r.name, err)
    }   
    r.setLastSyncResourceVersion(resourceVersion)

    resyncerrc := make(chan error, 1)
    go func() {
        for {
            select {
            case <-resyncCh:
            case <-stopCh:
                return
            }   
            glog.V(4).Infof("%s: next resync planned for %#v, forcing now", r.name, r.nextResync)
            if err := r.store.Resync(); err != nil {
                resyncerrc <- err 
                return
            }   
            cleanup()
            resyncCh, cleanup = r.resyncChan()
        }   
    }()
    
    for {
        timemoutseconds := int64(minWatchTimeout.Seconds() * (rand.Float64() + 1.0))
        options = api.ListOptions{
            ResourceVersion: resourceVersion,
            TimeoutSeconds: &timemoutseconds,
        }

        w, err := r.listerWatcher.Watch(options)
        ...
        if err := r.watchHandler(w, &resourceVersion, resyncerrc, stopCh); err != nil {
            if err != errorStopRequested {
                glog.Warningf("%s: watch of %v ended with: %v", r.name, r.expectedType, err)
            }
            return nil
        }
    }
}
```

这段代码其实，我们在分析watch server端机制时就介绍了 ，只是那时候是apiserver去watch etcd，而这里是kubelet去watch apiserver。这段代码被完全复用了。只是reflector的listerWatcher和cache不同。函数首先会进行List，然后开始Watch，最后启动一个watchHandler去处理watch到的东西。我们先看看List和Watch：

**pkg/client/cache/listwatch.go +**

```go
// List a set of apiserver resources
func (lw *ListWatch) List(options api.ListOptions) (runtime.Object, error) {
    return lw.ListFunc(options)
}

// Watch a set of apiserver resources
func (lw *ListWatch) Watch(options api.ListOptions) (watch.Interface, error) {
    return lw.WatchFunc(options)
}
```

可以看到他们调用ListFunc和WatchFunc去List和Watch apiserver的资源。分析ListFunc函数。

**pkg/client/cache/listwatch.go +49**

```go
func NewListWatchFromClient(c Getter, resource string, namespace string, fieldSelector fields.Selector) *ListWatch {
    listFunc := func(options api.ListOptions) (runtime.Object, error) {
        return c.Get().
            Namespace(namespace).
            Resource(resource).
            VersionedParams(&options, api.ParameterCodec).
            FieldsSelectorParam(fieldSelector).
            Do().
            Get()
    }   
    watchFunc := func(options api.ListOptions) (watch.Interface, error) {
        return c.Get().
            Prefix("watch").
            Namespace(namespace).
            Resource(resource).
            VersionedParams(&options, api.ParameterCodec).
            FieldsSelectorParam(fieldSelector).
            Watch()
    }   
    return &ListWatch{ListFunc: listFunc, WatchFunc: watchFunc}
}
```

首先Get()，说明需要发起一个Get请求，其实watchFunc也是发起一个Get请求。再看看Namespace(namespace)，其实只是在request设置namespace字段，再看看Resource函数,设置request的resource字段,VersionedParams主要对options进行序列化，options主要包括ResourceVersion和TimeoutSeconds这两个参数，FieldsSelectorParam函数主要将filter函数进行序列化，深入分析这个函数，你可以发现其实他将他序列化到一个嵌套的map里面。Do()函数发起真正的请求，并收到response，然后用r.transformResponse去处理response，包装成Result返回。Get()函数则主要对Result进行反序列化。最后返回结果。

现在来分析WatchFunc函数，这个函数很大一部分与ListFunc重合了,比如Get()、Namespace(namespace)、Resource(resource)VersionedParams(&options, api.ParameterCodec)、FieldsSelectorParam(fieldSelector)，有两个比较特殊的函数Prefix("watch")和Watch()。Prefix("watch")主要在pathPrefix的结尾增加了watch字段，用来和List请求区分（前面提过List和Watch都是作为Get请求发送出去）。Watch()函数重点分析一下。

**pkg/client/restclient/client.go +212**

```go
// Get begins a GET request. Short for c.Verb("GET").
func (c *RESTClient) Get() *Request {
    return c.Verb("GET") 
}
```

**pkg/client/restclient/request.go +233**

```go
func (r *Request) Namespace(namespace string) *Request {
    ...
    r.namespaceSet = true
    r.namespace = namespace
    return r
}

+174
func (r *Request) Resource(resource string) *Request {
    ...      
    r.resource = resource
    return r
}

+389
func (r *Request) FieldsSelectorParam(s fields.Selector) *Request {
    ...  
    s2, err := s.Transform(func(field, value string) (newField, newValue string, err error) {
        return fieldMappings.filterField(r.content.GroupVersion, r.resource, field, value)
    })
    ...   
    return r.setParam(unversioned.FieldSelectorQueryParam(r.content.GroupVersion.String()), s2.String())
}

+856
func (r *Request) Do() Result {
    r.tryThrottle()

    var result Result
    err := r.request(func(req *http.Request, resp *http.Response) {
        result = r.transformResponse(resp, req) 
    })   
    if err != nil {
        return Result{err: err} 
    }    
    return result
}

+1047
func (r Result) Get() (runtime.Object, error) { 
    if r.err != nil {    
        return nil, r.err
    }  
    if r.decoder == nil {
        return nil, fmt.Errorf("serializer for %s doesn't exist", r.contentType)
    }  
    return runtime.Decode(r.decoder, r.body)
}

+155
func (r *Request) Prefix(segments ...string) *Request {
    if r.err != nil {
        return r
    }    
    r.pathPrefix = path.Join(r.pathPrefix, path.Join(segments...))
    return r
}
```

**pkg/client/restclient/request.go +636**

```go
func (r *Request) Watch() (watch.Interface, error) {
    ...
    url := r.URL().String()
    req, err := http.NewRequest(r.verb, url, r.body)
    if err != nil {
        return nil, err
    }    
    req.Header = r.headers
    client := r.client
    if client == nil {
        client = http.DefaultClient
    }    
    r.backoffMgr.Sleep(r.backoffMgr.CalculateBackoff(r.URL()))
    resp, err := client.Do(req)
    updateURLMetrics(r, resp, err) 
    ...
    if resp.StatusCode != http.StatusOK {
        defer resp.Body.Close()
        if result := r.transformResponse(resp, req); result.err != nil {
            return nil, result.err
        }
        return nil, fmt.Errorf("for request '%+v', got status: %v", url, resp.StatusCode)
    }
    framer := r.serializers.Framer.NewFrameReader(resp.Body)
    decoder := streaming.NewDecoder(framer, r.serializers.StreamingSerializer)
    return watch.NewStreamWatcher(versioned.NewDecoder(decoder, r.serializers.Decoder)), nil
}
```

Watch()函数返回的并不是pods之类的具体资源，他返回的是watch.Interface，这个watch.Interface专门用来传送kubelet想要watch的资源。Watch首先会发起一个request，然后反序列化response，从response中获得watch.Interface。


获得watcher以后，reflector会调用r.watchHandler(w, &resourceVersion, resyncerrc, stopCh)去处理这个watcher

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

首先会从channel读取event，然后更新到r.store里面。r.store是什么呢？创建reflector的函数是cache.NewReflector(lw, &api.Pod{}, cache.NewUndeltaStore(send, cache.MetaNamespaceKeyFunc), 0) cache的类型为UndeltaStore，我们看看将event添加到这个cache里面会发生什么。

**pkg/client/cache/undelta_store.go**

```go
func (u *UndeltaStore) Add(obj interface{}) error {
    if err := u.Store.Add(obj); err != nil {
        return err
    }       
    u.PushFunc(u.Store.List())
    return nil
}           
                
func (u *UndeltaStore) Update(obj interface{}) error {
    if err := u.Store.Update(obj); err != nil {
        return err
    }           
    u.PushFunc(u.Store.List())
    return nil
}           
                
func (u *UndeltaStore) Delete(obj interface{}) error {
    if err := u.Store.Delete(obj); err != nil {
        return err
    }       
    u.PushFunc(u.Store.List())
    return nil  
} 
```
除了会将会操作实际的Store，还会执行u.PushFunc(u.Store.List())操作，PushFunc函数就是cache.NewUndeltaStore(send, cache.MetaNamespaceKeyFunc)里面的send函数，send函数我们前面提到过

**pkg/kubelet/config/apiserver.go +35**

```go
func newSourceApiserverFromLW(lw cache.ListerWatcher, updates chan<- interface{}) {
    send := func(objs []interface{}) {
        var pods []*api.Pod
        for _, o := range objs {
            pods = append(pods, o.(*api.Pod))
        }   
        updates <- kubetypes.PodUpdate{Pods: pods, Op: kubetypes.SET, Source: kubetypes.ApiserverSource}
    }   
    cache.NewReflector(lw, &api.Pod{}, cache.NewUndeltaStore(send, cache.MetaNamespaceKeyFunc), 0).Run()
}
```
主要是将List到的Pod发送到updates这个channel，kubelet从updates这个channel获取到Pod信息进行处理。这里我们可以发现无论是add、delete或者modify， u.PushFunc(u.Store.List())他会发送存储的所有pods。因为在这些操作之前，它都会先操作Store里面的pods对象，确保Store里面存储的是分配到该节点的Pod的最新信息。至于kubelet怎么处理获取的pods信息，这就不属于watch机制该讨论的范畴了。

## 总结

从上面的分析我们可以看出，reflector是一个重要的结构体，他的作用就是讲watch的对象存入一个Cache，并激发一些操作，apiserver对etcd进行watch时应用了这个代码，而kubelet对apiserver进行watch时也复用了这些代码。

